# discord-bot
"""
saachi Link Filter Bot (discord.py v2)
Features included:
- Powerful link detection (normalized, obfuscated, embed scanning, path-only)
- Bypass for Administrators, Server Owner, or role named "I see you"
- Per-guild enable/disable and per-channel disable/enable
- Welcome embed on join
- Warning ladder: 1..10 warnings -> kicks on 11 & 12 -> ban on 13
- DM + Public embed notifications (DM includes server name)
- Slash commands (app commands) + legacy prefix commands (m!)
- on_message_edit handler to catch delayed embeds/edited messages
- Persistent storage via SQLite (linkfilter.db)
"""

import re
import sqlite3
import unicodedata
from typing import Optional

import discord
from discord import app_commands
from discord.ext import commands
from discord.ui import Button, View

### ---------- CONFIG ----------
TOKEN = "code"
CLIENT_ID = "1407185051573293168"
PREFIX = "m!"
ROLE_BYPASS_NAME = "see you"   # Role that bypasses filter
DONATION_LINK = "https://yourdonatelink.example"
WELCOME_FOOTER = "Saachi link filter bot • Keeping your server safe 🔍"
DB_PATH = "linkfilter.db"
# -------------------------------

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.guilds = True

bot = commands.Bot(command_prefix=PREFIX, intents=intents)
tree = bot.tree  # for slash commands

### ---------- DB helpers ----------
def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""
    CREATE TABLE IF NOT EXISTS warnings (
        guild_id INTEGER,
        user_id INTEGER,
        count INTEGER,
        PRIMARY KEY (guild_id, user_id)
    )""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS guild_settings (
        guild_id INTEGER PRIMARY KEY,
        enabled INTEGER DEFAULT 1
    )""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS disabled_channels (
        guild_id INTEGER,
        channel_id INTEGER,
        PRIMARY KEY (guild_id, channel_id)
    )""")
    conn.commit()
    conn.close()

def get_warning(guild_id: int, user_id: int) -> int:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT count FROM warnings WHERE guild_id=? AND user_id=?", (guild_id, user_id))
    row = c.fetchone()
    conn.close()
    return row[0] if row else 0

def set_warning(guild_id: int, user_id: int, value: int):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute(
        "INSERT INTO warnings (guild_id, user_id, count) VALUES (?, ?, ?) "
        "ON CONFLICT(guild_id, user_id) DO UPDATE SET count=excluded.count",
        (guild_id, user_id, value)
    )
    conn.commit()
    conn.close()

def increment_warning(guild_id: int, user_id: int) -> int:
    cur = get_warning(guild_id, user_id) + 1
    set_warning(guild_id, user_id, cur)
    return cur

def guild_enabled(guild_id: int) -> bool:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT enabled FROM guild_settings WHERE guild_id=?", (guild_id,))
    row = c.fetchone()
    conn.close()
    if row is None:
        return True
    return bool(row[0])

def set_guild_enabled(guild_id: int, enabled: bool):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute(
        "INSERT INTO guild_settings (guild_id, enabled) VALUES (?, ?) "
        "ON CONFLICT(guild_id) DO UPDATE SET enabled=excluded.enabled",
        (guild_id, 1 if enabled else 0)
    )
    conn.commit()
    conn.close()

def is_channel_disabled(guild_id: int, channel_id: int) -> bool:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT 1 FROM disabled_channels WHERE guild_id=? AND channel_id=?", (guild_id, channel_id))
    row = c.fetchone()
    conn.close()
    return bool(row)

def set_channel_disabled(guild_id: int, channel_id: int, disabled: bool):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    if disabled:
        c.execute("INSERT OR IGNORE INTO disabled_channels (guild_id, channel_id) VALUES (?, ?)", (guild_id, channel_id))
    else:
        c.execute("DELETE FROM disabled_channels WHERE guild_id=? AND channel_id=?", (guild_id, channel_id))
    conn.commit()
    conn.close()

### ---------- Link Detection Logic ----------
COMMON_TLDS = [
    "com", "net", "org", "gg", "io", "ly", "xyz", "to", "link",
    "in", "app", "me", "cc", "tv"
]

ALIASES = {
    "dc.gg": "discord.gg",
    "discordgg": "discord.gg",
    "discord": "discord",
    "bitly": "bit.ly",
    "bit.ly": "bit.ly",
    "youtube": "youtube.com",
    "yt": "youtube.com",
    "gdrive": "drive.google.com",
    "share.google": "google.com"
}

URL_REGEX = re.compile(r"https?://[^\s<>]+", re.IGNORECASE)
WWW_REGEX = re.compile(r"www\.[^\s<>]+", re.IGNORECASE)
PATH_ONLY_REGEX = re.compile(r"\b[A-Za-z0-9_-]{3,}\/[A-Za-z0-9_-]{6,}\b")  # e.g. images/TblA0LPNzIFOKogwr

def normalize_text(s: str) -> str:
    # Unicode normalize to reduce confusables and remove control chars
    s = unicodedata.normalize("NFKD", s)
    # common obfuscations
    s = s.replace("(dot)", ".").replace("[dot]", ".").replace(" dot ", ".")
    s = s.replace("dot", ".")
    s = s.replace(":", ".")  # e.g., bit:ly -> bit.ly
    s = re.sub(r"\++", "", s)
    # remove spaces inside words like d i s c o r d
    s = re.sub(r"(?<=\w)\s+(?=\w)", "", s)
    # remove zero-width & non-printables
    s = "".join(ch for ch in s if unicodedata.category(ch)[0] != "C")
    return s.lower()

def looks_like_link(content: str) -> bool:
    if not content:
        return False
    s = normalize_text(content)
    # direct URL or www
    if URL_REGEX.search(s) or WWW_REGEX.search(s):
        return True
    # tld presence
    for tld in COMMON_TLDS:
        if f".{tld}" in s:
            return True
    # alias words (compact)
    compact = s.replace(" ", "")
    for alias in ALIASES.keys():
        if alias in compact:
            return True
    # detect discord.g.g.* like broken invites
    collapsed = re.sub(r"[^\w\.]", "", s)
    if "discord" in collapsed and "gg" in collapsed:
        return True
    # path-only links
    if PATH_ONLY_REGEX.search(s):
        return True
    # common shorteners
    if re.search(r"\b(bit\.ly|t\.co|tinyurl|is\.gd)\b", s):
        return True
    return False

### ---------- Moderation flow (embed messages & actions) ----------
async def take_action_for_link(message: discord.Message, reason_note: Optional[str] = None):
    guild = message.guild
    author = message.author
    if guild is None:
        return  # ignore DMs

    # delete offending message
    try:
        await message.delete()
    except Exception:
        pass

    # increment warning
    count = increment_warning(guild.id, author.id)

    # DM embed to the user (includes server name)
    dm_embed = discord.Embed(
        title="⚠️ Link Policy Violation",
        description=(
            f"You have violated the no-link policy in **{guild.name}**.\n\n"
            f"**Warning {count}/10** issued.\n\n"
            "After Warning 10 comes Kick 1 (11th), Kick 2 (12th), and Ban (13th).\n"
            "If you believe this was a mistake, please contact the server staff."
        ),
        color=discord.Color.orange()
    )
    dm_embed.set_footer(text=WELCOME_FOOTER)
    try:
        await author.send(embed=dm_embed)
    except Exception:
        # user might have DMs off
        pass

    # Public embed message visible to all (but not the offender-only)
    public_embed = discord.Embed(
        title="🚫 Forbidden Link Detected",
        description=f"{author.mention} tried to send a forbidden link.\n**Warning {count}/10** has been issued.",
        color=discord.Color.red()
    )
    public_embed.add_field(name="Note", value=f"Links are blocked unless you are Administrator or have the **{ROLE_BYPASS_NAME}** role.", inline=False)
    public_embed.set_footer(text=WELCOME_FOOTER)
    try:
        await message.channel.send(embed=public_embed)
    except Exception:
        pass

    # Punitive actions
    if count == 11 or count == 12:
        reason = f"Repeated link violations (warning {count})"
        try:
            await guild.kick(author, reason=reason)
            # kicked notification embed
            kicked_embed = discord.Embed(
                title="👢 User Kicked",
                description=f"{author} was kicked for repeated link violations (warning {count}).",
                color=discord.Color.dark_gold()
            )
            kicked_embed.set_footer(text=WELCOME_FOOTER)
            try:
                await message.channel.send(embed=kicked_embed)
            except:
                pass
            try:
                await author.send(embed=kicked_embed)
            except:
                pass
        except Exception:
            try:
                await message.channel.send("⚠️ Could not kick the user. Bot may lack permissions.")
            except:
                pass

    elif count >= 13:
        reason = f"Repeated link violations (warning {count})"
        try:
            await guild.ban(author, reason=reason)
            banned_embed = discord.Embed(
                title="⛔ User Banned",
                description=f"{author} was permanently banned for repeated link violations.",
                color=discord.Color.dark_red()
            )
            banned_embed.set_footer(text=WELCOME_FOOTER)
            try:
                await message.channel.send(embed=banned_embed)
            except:
                pass
            try:
                await author.send(embed=banned_embed)
            except:
                pass
        except Exception:
            try:
                await message.channel.send("⚠️ Could not ban the user. Bot may lack permissions.")
            except:
                pass

### ---------- Event handlers ----------
@bot.event
async def on_ready():
    print(f"Logged in as {bot.user} (ID: {bot.user.id})")
    try:
        await tree.sync()
        print("Slash commands synced.")
    except Exception as e:
        print("Could not sync slash commands:", e)
    init_db()

@bot.event
async def on_guild_join(guild: discord.Guild):
    # send welcome embed in the first channel we can send to
    channel = None
    for ch in guild.text_channels:
        perms = ch.permissions_for(guild.me)
        if perms.send_messages and perms.embed_links:
            channel = ch
            break
    if channel is None:
        return
    embed = discord.Embed(
        title="✅ Thanks for inviting me!",
        description=(
            "Hello! My name is **Saachi** 🤖\n\n"
            "My main job is to **block spam & suspicious links** in your server.\n\n"
            "➡️ Please keep my role **high in the role list** so I can see members and moderate properly.\n"
            f"➡️ I ignore Administrators, the Server Owner, and members with the **{ROLE_BYPASS_NAME}** role.\n\n"
            f"My command prefix is **{PREFIX}** and I also support slash commands (`/`).\n"
            f"Type **{PREFIX}helpme** or **/help** to see commands.\n\n"
            "If you need help, ask your server staff or contact the bot developer."
        ),
        color=discord.Color.blue()
    )
    embed.set_footer(text=WELCOME_FOOTER)
    # Invite and Donate buttons
    invite_url = f"https://discord.com/oauth2/authorize?client_id={CLIENT_ID}&permissions=268598080&scope=bot%20applications.commands"
    btn_invite = Button(label="➕ Invite Me", url=invite_url, style=discord.ButtonStyle.link)
    btn_donate = Button(label="💖 Support & Donate", url=DONATION_LINK, style=discord.ButtonStyle.link)
    view = View()
    view.add_item(btn_invite)
    view.add_item(btn_donate)
    try:
        await channel.send(embed=embed, view=view)
    except Exception:
        pass

@bot.event
async def on_message_edit(before: discord.Message, after: discord.Message):
    # Re-run the same check on edited message (to catch embeds that appear after)
    await on_message(after)

@bot.event
async def on_message(message: discord.Message):
    # ignore the bot's own messages
    if message.author.id == bot.user.id:
        return

    # process commands first (prefix commands)
    await bot.process_commands(message)

    guild = message.guild
    if guild is None:
        return  # ignore DMs

    # check global guild enable
    if not guild_enabled(guild.id):
        return

    # check per-channel disable state
    if is_channel_disabled(guild.id, message.channel.id):
        return

    # bypass if server owner
    try:
        if message.author == guild.owner:
            return
    except Exception:
        pass

    # bypass if author has admin perms
    try:
        if message.author.guild_permissions.administrator:
            return
    except Exception:
        pass

    # bypass if author has role bypass name
    try:
        role_bypass = discord.utils.get(message.author.roles, name=ROLE_BYPASS_NAME)
        if role_bypass:
            return
    except Exception:
        pass

    # Build combined text from message content + embed texts
    content = message.content or ""
    embed_texts = []
    for e in message.embeds:
        if getattr(e, "url", None):
            embed_texts.append(e.url)
        if getattr(e, "title", None):
            embed_texts.append(e.title)
        if getattr(e, "description", None):
            embed_texts.append(e.description)
        try:
            for f in e.fields:
                embed_texts.append(f.name or "")
                embed_texts.append(f.value or "")
        except Exception:
            pass

    combined = content + " " + " ".join(embed_texts)
    if looks_like_link(combined):
        await take_action_for_link(message)

### ---------- Commands (prefix + slash) ----------
def is_staff(member: discord.Member) -> bool:
    return (
        member.guild_permissions.manage_messages
        or member.guild_permissions.kick_members
        or member.guild_permissions.ban_members
        or member.guild_permissions.administrator
    )

# Helper to check admin OR owner
def is_admin_or_owner(member: discord.Member) -> bool:
    try:
        return member.guild_permissions.administrator or (member == member.guild.owner)
    except Exception:
        return False

# ----- Prefix commands -----
@bot.command(name="enable")
@commands.guild_only()
async def cmd_enable(ctx: commands.Context):
    # Require admin OR owner
    if not is_admin_or_owner(ctx.author):
        await ctx.send("❌ You must be an Administrator or the Server Owner to run this command.")
        return
    set_guild_enabled(ctx.guild.id, True)
    await ctx.send("✅ Link filter **enabled** for this server (all channels)")

@bot.command(name="disable")
@commands.guild_only()
async def cmd_disable(ctx: commands.Context):
    if not is_admin_or_owner(ctx.author):
        await ctx.send("❌ You must be an Administrator or the Server Owner to run this command.")
        return
    set_guild_enabled(ctx.guild.id, False)
    await ctx.send("⚠️ Link filter **disabled** for this server")

@bot.command(name="enable-channel")
@commands.guild_only()
async def cmd_enable_channel(ctx: commands.Context, channel: discord.TextChannel):
    if not is_admin_or_owner(ctx.author):
        await ctx.send("❌ You must be an Administrator or the Server Owner to run this command.")
        return
    set_channel_disabled(ctx.guild.id, channel.id, False)
    await ctx.send(f"✅ Link filter **enabled** in {channel.mention}")

@bot.command(name="disable-channel")
@commands.guild_only()
async def cmd_disable_channel(ctx: commands.Context, channel: discord.TextChannel):
    if not is_admin_or_owner(ctx.author):
        await ctx.send("❌ You must be an Administrator or the Server Owner to run this command.")
        return
    set_channel_disabled(ctx.guild.id, channel.id, True)
    await ctx.send(f"⚠️ Link filter **disabled** in {channel.mention}")

@bot.command(name="invite")
async def cmd_invite(ctx: commands.Context):
    invite_url = f"https://discord.com/oauth2/authorize?client_id={CLIENT_ID}&permissions=268598080&scope=bot%20applications.commands"
    btn = Button(label="➕ Invite Me", url=invite_url, style=discord.ButtonStyle.link)
    donate = Button(label="💖 Support", url=DONATION_LINK, style=discord.ButtonStyle.link)
    view = View()
    view.add_item(btn)
    view.add_item(donate)
    embed = discord.Embed(title="Invite saachi Link Filter Bot", description="Click the button to invite the bot to another server.", color=discord.Color.blue())
    await ctx.send(embed=embed, view=view)

@bot.command(name="helpme")
async def cmd_help(ctx: commands.Context):
    embed = discord.Embed(
        title="✨ Saachi Link Filter Bot — Commands",
        description=f"My prefix is **{PREFIX}** and I support slash commands (`/`).",
        color=discord.Color.purple()
    )
    embed.add_field(name="🔒 enable", value="Enable link filter for entire server (Admin/Owner)", inline=False)
    embed.add_field(name="🔓 disable", value="Disable link filter for entire server (Admin/Owner)", inline=False)
    embed.add_field(name="📌 enable-channel #channel", value="Enable filter in a specific channel (Admin/Owner)", inline=False)
    embed.add_field(name="📌 disable-channel #channel", value="Disable filter in a specific channel (Admin/Owner)", inline=False)
    embed.add_field(name="📝 warnings @user", value="View a user's warnings (Staff only)", inline=False)
    embed.set_footer(text=WELCOME_FOOTER)
    invite_url = f"https://discord.com/oauth2/authorize?client_id={CLIENT_ID}&permissions=268598080&scope=bot%20applications.commands"
    btn = Button(label="➕ Invite Me", url=invite_url, style=discord.ButtonStyle.link)
    donate = Button(label="💖 Support", url=DONATION_LINK, style=discord.ButtonStyle.link)
    view = View()
    view.add_item(btn)
    view.add_item(donate)
    await ctx.send(embed=embed, view=view)

@bot.command(name="donate")
async def cmd_donate(ctx: commands.Context):
    embed = discord.Embed(title="Support Development", description=f"Donate / Support: {DONATION_LINK}", color=discord.Color.gold())
    await ctx.send(embed=embed)

@bot.command(name="warnings")
@commands.guild_only()
async def cmd_warnings(ctx: commands.Context, member: discord.Member):
    if not is_staff(ctx.author):
        await ctx.send("❌ You don't have permission to view warnings.")
        return
    count = get_warning(ctx.guild.id, member.id)
    embed = discord.Embed(title="User Warnings", description=f"{member} has **{count}** warning(s) in this server.", color=discord.Color.blue())
    await ctx.send(embed=embed)

# ----- Slash commands -----
@tree.command(name="enable", description="Enable link filter for this server (Admin/Owner)")
@app_commands.checks.has_permissions(administrator=True)
async def slash_enable(interaction: discord.Interaction):
    # allow owner even if admin flag absent
    if interaction.guild is None:
        await interaction.response.send_message("This command can only be used in a server.", ephemeral=True)
        return
    if not (interaction.user.guild_permissions.administrator or interaction.user == interaction.guild.owner):
        await interaction.response.send_message("❌ You must be an Administrator or Server Owner.", ephemeral=True)
        return
    set_guild_enabled(interaction.guild.id, True)
    await interaction.response.send_message("✅ Link filter enabled for this server.", ephemeral=True)

@tree.command(name="disable", description="Disable link filter for this server (Admin/Owner)")
async def slash_disable(interaction: discord.Interaction):
    if interaction.guild is None:
        await interaction.response.send_message("This command can only be used in a server.", ephemeral=True)
        return
    if not (interaction.user.guild_permissions.administrator or interaction.user == interaction.guild.owner):
        await interaction.response.send_message("❌ You must be an Administrator or Server Owner.", ephemeral=True)
        return
    set_guild_enabled(interaction.guild.id, False)
    await interaction.response.send_message("⚠️ Link filter disabled for this server.", ephemeral=True)

@tree.command(name="enable-channel", description="Enable link filter in a specific channel (Admin/Owner)")
async def slash_enable_channel(interaction: discord.Interaction, channel: discord.TextChannel):
    if interaction.guild is None:
        await interaction.response.send_message("This command must be used in a server.", ephemeral=True)
        return
    if not (interaction.user.guild_permissions.administrator or interaction.user == interaction.guild.owner):
        await interaction.response.send_message("❌ You must be an Administrator or Server Owner.", ephemeral=True)
        return
    set_channel_disabled(interaction.guild.id, channel.id, False)
    await interaction.response.send_message(f"✅ Link filter enabled in {channel.mention}", ephemeral=True)

@tree.command(name="disable-channel", description="Disable link filter in a specific channel (Admin/Owner)")
async def slash_disable_channel(interaction: discord.Interaction, channel: discord.TextChannel):
    if interaction.guild is None:
        await interaction.response.send_message("This command must be used in a server.", ephemeral=True)
        return
    if not (interaction.user.guild_permissions.administrator or interaction.user == interaction.guild.owner):
        await interaction.response.send_message("❌ You must be an Administrator or Server Owner.", ephemeral=True)
        return
    set_channel_disabled(interaction.guild.id, channel.id, True)
    await interaction.response.send_message(f"⚠️ Link filter disabled in {channel.mention}", ephemeral=True)

@tree.command(name="invite", description="Get the bot invite link")
async def slash_invite(interaction: discord.Interaction):
    invite_url = f"https://discord.com/oauth2/authorize?client_id={CLIENT_ID}&permissions=268598080&scope=bot%20applications.commands"
    embed = discord.Embed(title="Invite Ultra Link Filter Bot", description="Click the button to invite the bot to another server.", color=discord.Color.blue())
    btn = Button(label="➕ Invite Me", url=invite_url, style=discord.ButtonStyle.link)
    donate = Button(label="💖 Support", url=DONATION_LINK, style=discord.ButtonStyle.link)
    view = View()
    view.add_item(btn)
    view.add_item(donate)
    await interaction.response.send_message(embed=embed, view=view, ephemeral=True)

@tree.command(name="help", description="Show help for the bot")
async def slash_help(interaction: discord.Interaction):
    embed = discord.Embed(title="✨ Ultra Link Filter Bot — Commands", description=f"My prefix is **{PREFIX}** and I support slash commands (`/`).", color=discord.Color.green())
    embed.add_field(name="/enable", value="Enable link filter for entire server (Admin/Owner)", inline=False)
    embed.add_field(name="/disable", value="Disable link filter for entire server (Admin/Owner)", inline=False)
    embed.add_field(name="/enable-channel", value="Enable filter in a specific channel (Admin/Owner)", inline=False)
    embed.add_field(name="/disable-channel", value="Disable filter in a specific channel (Admin/Owner)", inline=False)
    embed.add_field(name="/warnings", value="Check a user's warnings (Staff only)", inline=False)
    embed.set_footer(text=WELCOME_FOOTER)
    invite_url = f"https://discord.com/oauth2/authorize?client_id={CLIENT_ID}&permissions=268598080&scope=bot%20applications.commands"
    btn = Button(label="➕ Invite Me", url=invite_url, style=discord.ButtonStyle.link)
    donate = Button(label="💖 Support", url=DONATION_LINK, style=discord.ButtonStyle.link)
    view = View()
    view.add_item(btn)
    view.add_item(donate)
    await interaction.response.send_message(embed=embed, view=view, ephemeral=True)

@tree.command(name="warnings", description="View a user's warnings (Staff only)")
async def slash_warnings(interaction: discord.Interaction, member: discord.Member):
    if interaction.guild is None:
        await interaction.response.send_message("This command must be used in a server.", ephemeral=True)
        return
    if not is_staff(interaction.user):
        await interaction.response.send_message("❌ You don't have permission to view warnings.", ephemeral=True)
        return
    count = get_warning(interaction.guild.id, member.id)
    embed = discord.Embed(title="User Warnings", description=f"{member} has **{count}** warning(s) in this server.", color=discord.Color.blue())
    await interaction.response.send_message(embed=embed, ephemeral=True)

### ---------- Error handlers for commands ----------
@cmd_enable.error
@cmd_disable.error
@cmd_enable_channel.error
@cmd_disable_channel.error
async def admin_command_error(ctx, error):
    await ctx.send("You must be an Administrator or the Server Owner to run this command.")

# Start the bot
if __name__ == "__main__":
    init_db()
    bot.run(TOKEN)
