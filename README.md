<div align="center">
  <h1>Python Developer</h1>
  <a href="https://www.python.org" target="_blank">
    <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/python/python-original.svg" alt="python" width="150" height="150"/>
  </a>
</div>

```python
import discord
import os
from dotenv import load_dotenv


intents = discord.Intents.default()
intents.message_content = True
intents.members = True


bot = discord.Bot(
    intents=intents
)


@bot.event
async def on_ready():
    print(f"{bot.user} ist online!")


if __name__ == "__main__":
    for filename in os.listdir("cogs"):
        if filename.endswith(".py"):
            bot.load_extension(f"cogs.{filename[:-3]}")


load_dotenv()
bot.run(os.getenv("TOKEN"))
```
# kontakt@keks-app.de

