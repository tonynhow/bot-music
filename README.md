import discord
from discord.ext import commands
import yt_dlp
import asyncio
import os
import re

# Cria pasta de downloads se não existir
os.makedirs("downloads", exist_ok=True)

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix=".", intents=intents)

queue = []

def is_url(string):
    return re.match(r'https?://', string) is not None

def is_playing(vc):
    return vc and (vc.is_playing() or vc.is_paused())

def clean_downloads_folder():
    files = sorted(
        [f for f in os.listdir("downloads") if f.endswith(".mp3")],
        key=lambda x: os.path.getctime(os.path.join("downloads", x))
    )
    for file in files[:-3]:
        os.remove(os.path.join("downloads", file))

async def download_music(query):
    ydl_opts = {
        "format": "bestaudio/best",
        "noplaylist": True,
        "outtmpl": "downloads/%(title)s.%(ext)s",
        "postprocessors": [{
            "key": "FFmpegExtractAudio",
            "preferredcodec": "mp3",
            "preferredquality": "192"
        }],
        "quiet": True
    }

    if not is_url(query):
        ydl_opts["default_search"] = "ytsearch1"

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(query, download=True)
            title = info.get("title", "Música")
            for f in os.listdir("downloads"):
                if f.endswith(".mp3"):
                    return os.path.join("downloads", f), title
    except Exception as e:
        print(f"[ERRO] Falha ao baixar: {e}")
        return None, str(e)
    return None, "Erro desconhecido"

@bot.event
async def on_ready():
    print(f"🤖 Bot online como {bot.user}")

@bot.command()
async def join(ctx):
    if ctx.author.voice:
        channel = ctx.author.voice.channel
        if ctx.voice_client:
            await ctx.voice_client.move_to(channel)
        else:
            await channel.connect()
        await ctx.send(f"✅ Entrei em: **{channel.name}**")
    else:
        await ctx.send("🚫 Você precisa estar em uma call primeiro!")

@bot.command()
async def play(ctx, *, search: str):
    vc = ctx.voice_client
    if not vc:
        if ctx.author.voice:
            vc = await ctx.author.voice.channel.connect()
        else:
            return await ctx.send("🚫 Você precisa estar em uma call primeiro!")

    path, title = await download_music(search)
    if not path:
        return await ctx.send(f"❌ Erro ao baixar: {title}")

    queue.append((path, title))
    clean_downloads_folder()

    if not is_playing(vc):
        await tocar_proxima(ctx)

async def tocar_proxima(ctx):
    if not queue:
        await ctx.send("📭 Fila vazia. Saindo do canal.")
        if ctx.voice_client:
            await ctx.voice_client.disconnect()
        return

    path, title = queue.pop(0)
    source = discord.FFmpegPCMAudio(path)

    def depois(err):
        if err:
            print(f"[ERRO] Erro ao tocar: {err}")
        fut = tocar_proxima(ctx)
        asyncio.run_coroutine_threadsafe(fut, bot.loop)

    ctx.voice_client.play(source, after=depois)
    await ctx.send(f"🎵 Tocando agora: **{title}**")

@bot.command()
async def skip(ctx):
    if is_playing(ctx.voice_client):
        ctx.voice_client.stop()
        await ctx.send("⏩ Pulando música...")
    else:
        await ctx.send("🚫 Nenhuma música tocando.")

@bot.command()
async def pause(ctx):
    if is_playing(ctx.voice_client):
        ctx.voice_client.pause()
        await ctx.send("⏸️ Música pausada.")
    else:
        await ctx.send("🚫 Nenhuma música tocando.")

@bot.command()
async def resume(ctx):
    if ctx.voice_client and ctx.voice_client.is_paused():
        ctx.voice_client.resume()
        await ctx.send("▶️ Música retomada.")
    else:
        await ctx.send("🚫 Nenhuma música pausada.")

@bot.command()
async def leave(ctx):
    if ctx.voice_client:
        await ctx.voice_client.disconnect()
        await ctx.send("👋 Saí do canal de voz.")
    else:
        await ctx.send("🚫 Nem estou na call.")

# RODAR O BOT
bot.run(
