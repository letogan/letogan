# Python Developer

<p align="center">
  <a href="https://www.python.org" target="_blank" rel="noreferrer">
    <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/python/python-original.svg" alt="Python" width="100" height="100"/>
  </a>
</p>

# Beispiel:

```python
import discord
from discord.commands import slash_command
from discord.ext import commands, tasks
import asyncio
import random
from datetime import timedelta
import datetime
import json


class GiveawayView(discord.ui.View):
    def __init__(self, cog, giveaway_data):
        super().__init__(timeout=None)
        self.cog = cog
        self.giveaway_data = giveaway_data

    @discord.ui.button(label="Teilnehmen", style=discord.ButtonStyle.green, custom_id="giveaway_join")
    async def join_giveaway(self, button: discord.ui.Button, interaction: discord.Interaction):
        if interaction.user.id not in self.giveaway_data['participants']:
            self.giveaway_data['participants'].append(interaction.user.id)
            await interaction.response.send_message("Du nimmst **nun** am Giveaway teil!", ephemeral=True, delete_after=10)
        else:
            await interaction.response.send_message("Du nimmst **bereits** am Giveaway teil!", ephemeral=True, delete_after=10)

        channel = self.cog.bot.get_channel(self.giveaway_data['channel_id'])
        if not channel:
            print(f"Kanal mit ID {self.giveaway_data['channel_id']} nicht gefunden.")
            return

        try:
            message = await channel.fetch_message(self.giveaway_data['message_id'])
            embed = message.embeds[0]
            embed.set_field_at(2, name="🏷️ Teilnehmer", value=str(len(self.giveaway_data['participants'])), inline=True)
            await message.edit(embed=embed)
        except discord.NotFound:
            print(f"Nachricht mit ID {self.giveaway_data['message_id']} nicht gefunden.")
        except Exception as e:
            print(f"Fehler beim Aktualisieren des Giveaways: {e}")

        self.cog.giveaways[str(message.id)] = self.giveaway_data
        self.cog.save_giveaways()

    async def disable_buttons(self):
        """Deaktiviert die Buttons nach Ende des Giveaways."""
        for child in self.children:
            child.disabled = True


class Giveaway(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.giveaways = {}
        self.load_giveaways()
        self.check_giveaways.start()

    def cog_unload(self):
        self.check_giveaways.cancel()

    def load_giveaways(self):
        try:
            with open('giveaways.json', 'r') as f:
                self.giveaways = json.load(f)
        except FileNotFoundError:
            self.giveaways = {}

    def save_giveaways(self):
        with open('giveaways.json', 'w') as f:
            json.dump(self.giveaways, f)

    @tasks.loop(minutes=1)
    async def check_giveaways(self):
        current_time = datetime.datetime.utcnow()
        to_remove = []

        for giveaway_id, giveaway_data in self.giveaways.items():
            end_time = datetime.datetime.fromisoformat(giveaway_data['end_time'])
            if current_time >= end_time:
                await self.end_giveaway(giveaway_id, giveaway_data)
                to_remove.append(giveaway_id)

        for giveaway_id in to_remove:
            del self.giveaways[giveaway_id]

        self.save_giveaways()

    async def end_giveaway(self, giveaway_id, giveaway_data):
        channel = self.bot.get_channel(giveaway_data['channel_id'])
        if not channel:
            print(f"Kanal mit ID {giveaway_data['channel_id']} nicht gefunden.")
            return

        try:
            message = await channel.fetch_message(int(giveaway_id))
            if message:
                if giveaway_data['participants']:
                    winners = random.sample(
                        giveaway_data['participants'],
                        min(giveaway_data['winners'], len(giveaway_data['participants']))
                    )
                    winner_mentions = [f"<@{winner_id}>" for winner_id in winners]
                    await message.reply(f"**Das Giveaway ist beendet!**\n 🎊 __**Gewinner:**__ {', '.join(winner_mentions)}")
                else:
                    await message.reply("Das Giveaway ist beendet, aber es gab keine Teilnehmer.")

                view = GiveawayView(self, giveaway_data)
                await view.disable_buttons()
                await message.edit(view=view)

                embed = message.embeds[0]

                embed.remove_field(3)

                embed.add_field(name="❌ Giveaway beendet", value="Melde dich bei Fragen im Support.", inline=False)
                await message.edit(embed=embed)
        except discord.NotFound:
            print(f"Nachricht mit ID {giveaway_id} nicht gefunden.")
        except Exception as e:
            print(f"Fehler beim Beenden des Giveaways: {e}")

    @commands.Cog.listener()
    async def on_ready(self):
        for giveaway_id, giveaway_data in self.giveaways.items():
            self.bot.add_view(GiveawayView(self, giveaway_data))

    @slash_command(description="🎁︱Starte ein Giveaway")
    @discord.guild_only()
    @discord.default_permissions(manage_messages=True)
    async def giveaway(self, ctx: discord.ApplicationContext, preis: str, gewinner: int, dauer: int, einheit: str):
        if einheit.lower() == "minuten":
            end_time = datetime.datetime.utcnow() + timedelta(minutes=dauer)
        elif einheit.lower() == "stunden":
            end_time = datetime.datetime.utcnow() + timedelta(hours=dauer)
        elif einheit.lower() == "tage":
            end_time = datetime.datetime.utcnow() + timedelta(days=dauer)
        else:
            await ctx.respond("Ungültige Zeiteinheit! Bitte nutze **Minuten**, **Stunden** oder **Tage**.", ephemeral=True, delete_after=10)
            return

        end_timestamp = int(end_time.replace(tzinfo=datetime.timezone.utc).timestamp())

        embed = discord.Embed(title="Giveaway", color=0xEDAB54)
        embed.set_author(name=ctx.user.display_name, icon_url=ctx.user.display_avatar.url)
        embed.add_field(name="🎁 Preis", value=f"{preis}", inline=False)
        embed.add_field(name="🎉 Gewinner", value=f"{gewinner}", inline=False)
        embed.add_field(name="🏷️ Teilnehmer", value="0", inline=False)
        embed.add_field(name="⏳ Endet in", value=f"<t:{end_timestamp}:R>", inline=False)
        embed.set_footer(text="Klicke auf den Button, um teilzunehmen!")

        giveaway_data = {
            'prize': preis,
            'winners': gewinner,
            'end_time': end_time.isoformat(),
            'participants': [],
            'channel_id': ctx.channel.id
        }

        view = GiveawayView(self, giveaway_data)
        message = await ctx.respond(embed=embed, view=view)
        message = await message.original_response()

        giveaway_data['message_id'] = message.id
        self.giveaways[str(message.id)] = giveaway_data
        self.save_giveaways()


def setup(bot):
    bot.add_cog(Giveaway(bot))
