import os
import discord
from discord.ext import commands
import asyncio

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix="+", intents=intents)

# Dictionnaire pour suivre les utilisateurs mutés et la durée restante de leur mute
muted_users = {}

token = os.environ['TOKEN_BOT_DISCORD']

@bot.event
async def on_ready():
    print(f'Logged in as {bot.user}!')

@bot.command()
async def mute(ctx, member: discord.Member, time: str, *, reason: str = None):
    # Vérifier si l'utilisateur est déjà mute
    if member.id in muted_users:
        # Récupérer le temps restant du mute
        remaining_time = muted_users[member.id]['time']
        await ctx.send(f"{member.mention} est déjà mute pendant {remaining_time}m. Veuillez attendre la fin de son mute actuel.")
        return

    # Vérifier que le temps est bien 5m, 10m ou 15m
    if time not in ["5m", "10m", "15m"]:
        await ctx.send("Temps invalide. Utilise 5m, 10m ou 15m.")
        return

    # Vérifier si une raison est fournie
    if not reason:
        await ctx.send("Tu dois fournir une raison pour le mute.")
        return

    # Convertir le temps en minutes
    mute_duration = int(time[:-1])  # Retirer 'm' pour obtenir le nombre de minutes

    # Ajouter le rôle 'mute' si il n'existe pas
    role = discord.utils.get(ctx.guild.roles, name="mute")
    if not role:
        # Créer un rôle 'mute' s'il n'existe pas
        role = await ctx.guild.create_role(name="mute", reason="Rôle pour muter les utilisateurs")
        # On s'assure que ce rôle a les bonnes permissions
        await role.edit(permissions=discord.Permissions(send_messages=False, speak=False))

    # Sauvegarder les rôles actuels de l'utilisateur
    original_roles = member.roles[1:]  # On garde tous les rôles sauf @everyone

    # Ajouter le rôle mute et supprimer tous les autres rôles
    await member.add_roles(role)
    await member.edit(roles=[role])

    # Ajouter l'utilisateur à la liste des utilisateurs mutés avec le temps restant
    muted_users[member.id] = {
        'time': mute_duration,
        'member': member
    }

    # Message pour indiquer que l'utilisateur est mute
    await ctx.send(f"{member.mention} a été mute pendant {mute_duration}m pour : {reason}.")

    # Attendre la durée du mute (en secondes)
    await asyncio.sleep(mute_duration * 60)

    # Retirer le rôle mute et restaurer les rôles originaux
    await member.edit(roles=original_roles)
    await member.remove_roles(role)

    # Annoncer que l'utilisateur a été unmute
    await ctx.send(f"{member.mention} tu as été demute.")

    # Retirer l'utilisateur de la liste des utilisateurs mutés
    del muted_users[member.id]

bot.run(token)
