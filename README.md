import discord
from discord import app_commands
from discord.ext import commands
import os
from datetime import datetime, timedelta

intents = discord.Intents.default()
intents.message_content = True
intents.guilds = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)

# In-memory storage (for demo)
authorized_roles = set()
logs = []
infractions = []

def is_authorized(interaction: discord.Interaction):
    return any(role.id in authorized_roles for role in interaction.user.roles)

@bot.event
async def on_ready():
    await bot.tree.sync()
    print(f"Bot is ready. Logged in as {bot.user}")

@bot.tree.command(name="setup")
@app_commands.describe(role="Role authorized to use bot commands")
async def setup(interaction: discord.Interaction, role: discord.Role):
    authorized_roles.add(role.id)
    await interaction.response.send_message(f"Role {role.name} authorized.", ephemeral=True)

@bot.tree.command(name="infraction")
@app_commands.describe(user="User to infraction", reason="Reason for the infraction")
async def infraction(interaction: discord.Interaction, user: discord.Member, reason: str):
    if not is_authorized(interaction):
        await interaction.response.send_message("You are not authorized to use this command.", ephemeral=True)
        return
    infraction_id = len(infractions) + 1
    record = {
        "id": infraction_id,
        "officer": interaction.user.id,
        "user": user.id,
        "reason": reason,
        "timestamp": datetime.utcnow()
    }
    infractions.append(record)
    await interaction.response.send_message(f"Infraction #{infraction_id} issued to {user.mention}.", ephemeral=True)

@bot.tree.command(name="revoke_infraction")
@app_commands.describe(infraction_id="ID of the infraction to revoke")
async def revoke_infraction(interaction: discord.Interaction, infraction_id: int):
    if not is_authorized(interaction):
        await interaction.response.send_message("Unauthorized.", ephemeral=True)
        return
    global infractions
    infractions = [i for i in infractions if i["id"] != infraction_id]
    await interaction.response.send_message(f"Infraction #{infraction_id} revoked.", ephemeral=True)

@bot.tree.command(name="view_infractions")
@app_commands.describe(user="User whose infractions to view")
async def view_infractions(interaction: discord.Interaction, user: discord.Member):
    user_infractions = [i for i in infractions if i["user"] == user.id]
    if not user_infractions:
        await interaction.response.send_message(f"No infractions for {user.display_name}.", ephemeral=True)
        return
    msg = "\n".join([f"#{i['id']} - {i['reason']} on {i['timestamp'].strftime('%Y-%m-%d')}" for i in user_infractions])
    await interaction.response.send_message(f"Infractions for {user.display_name}:\n{msg}", ephemeral=True)

@bot.tree.command(name="arrest")
@app_commands.describe(suspect="Suspect name", location="Location", reason="What happened", mugshot="Upload a mugshot")
async def arrest(interaction: discord.Interaction, suspect: str, location: str, reason: str, mugshot: discord.Attachment):
    if not is_authorized(interaction):
        await interaction.response.send_message("Unauthorized.", ephemeral=True)
        return
    logs.append({"type": "arrest", "officer": interaction.user.id, "suspect": suspect, "location": location, "reason": reason, "image": mugshot.url, "timestamp": datetime.utcnow()})
    await interaction.response.send_message(f"Arrest logged for {suspect}.", ephemeral=True)

@bot.tree.command(name="citation")
@app_commands.describe(suspect="Suspect name", location="Location", reason="What happened", citation_pic="Upload the citation")
async def citation(interaction: discord.Interaction, suspect: str, location: str, reason: str, citation_pic: discord.Attachment):
    if not is_authorized(interaction):
        await interaction.response.send_message("Unauthorized.", ephemeral=True)
        return
    logs.append({"type": "citation", "officer": interaction.user.id, "suspect": suspect, "location": location, "reason": reason, "image": citation_pic.url, "timestamp": datetime.utcnow()})
    await interaction.response.send_message(f"Citation logged for {suspect}.", ephemeral=True)

@bot.tree.command(name="statistics")
@app_commands.describe(user="User to get stats for", days="How many days back to check")
async def statistics(interaction: discord.Interaction, user: discord.Member, days: int):
    cutoff = datetime.utcnow() - timedelta(days=days)
    user_logs = [log for log in logs if log["officer"] == user.id and log["timestamp"] >= cutoff]
    await interaction.response.send_message(f"{user.display_name} has logged {len(user_logs)} actions in the past {days} days.", ephemeral=True)

@bot.tree.command(name="statistics_department")
@app_commands.describe(days="How many days back to check")
async def statistics_department(interaction: discord.Interaction, days: int):
    cutoff = datetime.utcnow() - timedelta(days=days)
    count = sum(1 for log in logs if log["timestamp"] >= cutoff)
    await interaction.response.send_message(f"The department has logged {count} actions in the past {days} days.", ephemeral=True)

# Run bot
bot.run(os.getenv("DISCORD_TOKEN"))
