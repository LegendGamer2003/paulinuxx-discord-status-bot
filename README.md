from pathlib import Path
import textwrap, zipfile, json, os

root = Path("/mnt/data/paulinuxx-discord-status-bot")
(root / "src" / "commands").mkdir(parents=True, exist_ok=True)
(root / "src" / "events").mkdir(parents=True, exist_ok=True)
(root / "src" / "utils").mkdir(parents=True, exist_ok=True)
(root / "data").mkdir(parents=True, exist_ok=True)

files = {
"package.json": r'''{
  "name": "paulinuxx-discord-status-bot",
  "version": "1.0.0",
  "description": "PAULINUXX Discord server management and statistics bot",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js",
    "deploy": "node src/deploy-commands.js"
  },
  "dependencies": {
    "discord.js": "^14.22.1",
    "dotenv": "^17.2.1"
  }
}''',

".env.example": r'''DISCORD_TOKEN=PASTE_NEW_BOT_TOKEN_HERE
CLIENT_ID=YOUR_BOT_APPLICATION_ID
GUILD_ID=YOUR_SERVER_ID

# Optional role/channel IDs for reports and tickets
STAFF_ROLE_ID=
TICKET_CATEGORY_ID=
LOG_CHANNEL_ID=
REPORT_CHANNEL_ID=
''',

".gitignore": r'''node_modules/
.env
data/*.json
*.log
''',

"src/utils/config.js": r'''require("dotenv").config();

module.exports = {
  token: process.env.DISCORD_TOKEN,
  clientId: process.env.CLIENT_ID,
  guildId: process.env.GUILD_ID,
  staffRoleId: process.env.STAFF_ROLE_ID || null,
  ticketCategoryId: process.env.TICKET_CATEGORY_ID || null,
  logChannelId: process.env.LOG_CHANNEL_ID || null,
  reportChannelId: process.env.REPORT_CHANNEL_ID || null
};
''',

"src/utils/stats.js": r'''const { EmbedBuilder } = require("discord.js");

function getStatsEmbed(guild) {
  const members = guild.memberCount;
  const bots = guild.members.cache.filter(m => m.user.bot).size;
  const humans = Math.max(0, members - bots);
  const online = guild.members.cache.filter(m => m.presence?.status && m.presence.status !== "offline").size;

  return new EmbedBuilder()
    .setTitle("📊 PAULINUXX SERVER STATISTIKA")
    .setDescription("Automatiškai atnaujinama serverio statistika.")
    .addFields(
      { name: "👥 Visi nariai", value: `**${members}**`, inline: true },
      { name: "🟢 Online", value: `**${online}**`, inline: true },
      { name: "👤 Žmonės", value: `**${humans}**`, inline: true },
      { name: "🤖 Botai", value: `**${bots}**`, inline: true },
      { name: "💬 Kanalai", value: `**${guild.channels.cache.size}**`, inline: true },
      { name: "🛡️ Rolės", value: `**${guild.roles.cache.size}**`, inline: true }
    )
    .setFooter({ text: "PAULINUXX • Statistics" })
    .setTimestamp();
}

module.exports = { getStatsEmbed };
''',

"src/utils/logger.js": r'''const { EmbedBuilder } = require("discord.js");
const config = require("./config");

async function log(guild, title, description, fields = []) {
  if (!config.logChannelId) return;
  const channel = guild.channels.cache.get(config.logChannelId);
  if (!channel?.isTextBased()) return;

  const embed = new EmbedBuilder()
    .setTitle(title)
    .setDescription(description)
    .addFields(fields)
    .setTimestamp();

  await channel.send({ embeds: [embed] }).catch(() => {});
}

module.exports = { log };
''',

"src/utils/tickets.js": r'''const {
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  ChannelType,
  EmbedBuilder,
  PermissionsBitField
} = require("discord.js");
const config = require("./config");

async function createTicket(interaction, type = "Pagalba") {
  const existing = interaction.guild.channels.cache.find(
    c => c.type === ChannelType.GuildText &&
         c.topic === `ticket-owner:${interaction.user.id}`
  );

  if (existing) {
    return interaction.reply({
      content: `❌ Jau turi atidarytą ticket: ${existing}`,
      ephemeral: true
    });
  }

  const safeName = interaction.user.username
    .toLowerCase()
    .replace(/[^a-z0-9-]/g, "")
    .slice(0, 18) || "user";

  const overwrites = [
    {
      id: interaction.guild.roles.everyone.id,
      deny: [PermissionsBitField.Flags.ViewChannel]
    },
    {
      id: interaction.user.id,
      allow: [
        PermissionsBitField.Flags.ViewChannel,
        PermissionsBitField.Flags.SendMessages,
        PermissionsBitField.Flags.ReadMessageHistory
      ]
    }
  ];

  if (config.staffRoleId) {
    overwrites.push({
      id: config.staffRoleId,
      allow: [
        PermissionsBitField.Flags.ViewChannel,
        PermissionsBitField.Flags.SendMessages,
        PermissionsBitField.Flags.ReadMessageHistory
      ]
    });
  }

  const channel = await interaction.guild.channels.create({
    name: `ticket-${safeName}`,
    type: ChannelType.GuildText,
    parent: config.ticketCategoryId || undefined,
    topic: `ticket-owner:${interaction.user.id}`,
    permissionOverwrites: overwrites
  });

  const embed = new EmbedBuilder()
    .setTitle(`🎫 ${type}`)
    .setDescription(
      `Sveikas, ${interaction.user}!\n\n` +
      `Aprašyk savo problemą kuo išsamiau. Staff narys netrukus atsakys.\n\n` +
      `**Ticket tipas:** ${type}`
    )
    .setTimestamp();

  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder()
      .setCustomId("ticket_close")
      .setLabel("Uždaryti")
      .setEmoji("🔒")
      .setStyle(ButtonStyle.Danger)
  );

  await channel.send({ embeds: [embed], components: [row] });
  return interaction.reply({ content: `✅ Ticket sukurtas: ${channel}`, ephemeral: true });
}

module.exports = { createTicket };
''',

"src/commands/setup.js": r'''const {
  SlashCommandBuilder,
  PermissionFlagsBits,
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle
} = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("setup")
    .setDescription("Sukuria pagrindinį PAULINUXX support panel")
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  async execute(interaction) {
    const embed = new EmbedBuilder()
      .setTitle("🎫 PAULINUXX PAGALBA")
      .setDescription(
        "Pasirinkite, kokios pagalbos reikia.\n\n" +
        "🎫 **Support** – bendra pagalba\n" +
        "🐛 **Bug Report** – pranešti apie klaidą\n" +
        "👤 **Player Report** – pranešti apie žaidėją\n" +
        "📝 **Staff Application** – pateikti paraišką"
      )
      .setFooter({ text: "PAULINUXX • Support System" });

    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId("ticket_support").setLabel("Support").setEmoji("🎫").setStyle(ButtonStyle.Primary),
      new ButtonBuilder().setCustomId("ticket_bug").setLabel("Bug Report").setEmoji("🐛").setStyle(ButtonStyle.Secondary),
      new ButtonBuilder().setCustomId("ticket_player").setLabel("Player Report").setEmoji("👤").setStyle(ButtonStyle.Danger),
      new ButtonBuilder().setCustomId("ticket_staff").setLabel("Staff Application").setEmoji("📝").setStyle(ButtonStyle.Success)
    );

    await interaction.channel.send({ embeds: [embed], components: [row] });
    await interaction.reply({ content: "✅ Support panel sukurtas.", ephemeral: true });
  }
};
''',

"src/commands/stats.js": r'''const { SlashCommandBuilder } = require("discord.js");
const { getStatsEmbed } = require("../utils/stats");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("stats")
    .setDescription("Parodo serverio statistiką"),

  async execute(interaction) {
    await interaction.reply({ embeds: [getStatsEmbed(interaction.guild)] });
  }
};
''',

"src/commands/ping.js": r'''const { SlashCommandBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("ping")
    .setDescription("Patikrina boto veikimą"),

  async execute(interaction) {
    await interaction.reply(`🏓 Pong! API: **${interaction.client.ws.ping}ms**`);
  }
};
''',

"src/commands/help.js": r'''const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("help")
    .setDescription("Parodo boto komandas"),

  async execute(interaction) {
    const embed = new EmbedBuilder()
      .setTitle("🤖 PAULINUXX BOT")
      .setDescription("Pagrindinės komandos:")
      .addFields(
        { name: "/ping", value: "Patikrinti boto ping.", inline: true },
        { name: "/stats", value: "Serverio statistika.", inline: true },
        { name: "/setup", value: "Support panel. Tik administratoriams.", inline: true }
      );
    await interaction.reply({ embeds: [embed], ephemeral: true });
  }
};
''',

"src/index.js": r'''const {
  Client,
  Collection,
  GatewayIntentBits,
  Partials,
  ActivityType,
  Events
} = require("discord.js");
const fs = require("fs");
const path = require("path");
const config = require("./utils/config");
const { getStatsEmbed } = require("./utils/stats");
const { createTicket } = require("./utils/tickets");
const { log } = require("./utils/logger");

if (!config.token) {
  console.error("❌ DISCORD_TOKEN nerastas .env faile.");
  process.exit(1);
}

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildPresences,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ],
  partials: [Partials.Channel]
});

client.commands = new Collection();

const commandsPath = path.join(__dirname, "commands");
for (const file of fs.readdirSync(commandsPath).filter(f => f.endsWith(".js"))) {
  const command = require(path.join(commandsPath, file));
  client.commands.set(command.data.name, command);
}

client.once(Events.ClientReady, async c => {
  console.log(`✅ Prisijungta kaip ${c.user.tag}`);

  c.user.setPresence({
    activities: [{ name: "PAULINUXX serverį", type: ActivityType.Watching }],
    status: "online"
  });

  setInterval(async () => {
    for (const guild of c.guilds.cache.values()) {
      const statsChannel = guild.channels.cache.find(
        ch => ch.isTextBased() && ch.name.toLowerCase().includes("server-stat")
      );
      if (!statsChannel) continue;

      const messages = await statsChannel.messages.fetch({ limit: 20 }).catch(() => null);
      if (!messages) continue;

      const botMessage = messages.find(m =>
        m.author.id === c.user.id &&
        m.embeds[0]?.title === "📊 PAULINUXX SERVER STATISTIKA"
      );

      if (botMessage) {
        await botMessage.edit({ embeds: [getStatsEmbed(guild)] }).catch(() => {});
      }
    }
  }, 30000);
});

client.on(Events.InteractionCreate, async interaction => {
  try {
    if (interaction.isChatInputCommand()) {
      const command = client.commands.get(interaction.commandName);
      if (!command) return;
      await command.execute(interaction);
      return;
    }

    if (interaction.isButton()) {
      const map = {
        ticket_support: "Support",
        ticket_bug: "Bug Report",
        ticket_player: "Player Report",
        ticket_staff: "Staff Application"
      };

      if (map[interaction.customId]) {
        await createTicket(interaction, map[interaction.customId]);
        await log(
          interaction.guild,
          "🎫 Naujas ticket",
          `${interaction.user} sukūrė **${map[interaction.customId]}** ticket.`
        );
        return;
      }

      if (interaction.customId === "ticket_close") {
        await interaction.reply({ content: "🔒 Ticket uždaromas...", ephemeral: true });
        setTimeout(() => interaction.channel.delete().catch(() => {}), 1500);
      }
    }
  } catch (error) {
    console.error(error);
    const reply = {
      content: "❌ Įvyko klaida. Patikrink konsolę.",
      ephemeral: true
    };
    if (interaction.replied || interaction.deferred) {
      await interaction.followUp(reply).catch(() => {});
    } else {
      await interaction.reply(reply).catch(() => {});
    }
  }
});

client.on(Events.GuildMemberAdd, member => {
  log(member.guild, "👋 Naujas narys", `${member.user} prisijungė prie serverio.`);
});

client.on(Events.GuildMemberRemove, member => {
  log(member.guild, "👋 Narys išėjo", `${member.user.tag} paliko serverį.`);
});

client.login(config.token);
''',

"src/deploy-commands.js": r'''const { REST, Routes } = require("discord.js");
const fs = require("fs");
const path = require("path");
const config = require("./utils/config");

const commands = [];
const commandsPath = path.join(__dirname, "commands");

for (const file of fs.readdirSync(commandsPath).filter(f => f.endsWith(".js"))) {
  const command = require(path.join(commandsPath, file));
  commands.push(command.data.toJSON());
}

const rest = new REST({ version: "10" }).setToken(config.token);

(async () => {
  try {
    console.log(`🔄 Registruojamos ${commands.length} komandos...`);
    await rest.put(
      Routes.applicationGuildCommands(config.clientId, config.guildId),
      { body: commands }
    );
    console.log("✅ Slash komandos užregistruotos.");
  } catch (error) {
    console.error(error);
  }
})();
''',

"README.md": r'''# PAULINUXX Discord Status Bot

Discord.js v14 botas su serverio statistika, slash komandomis, support/ticket sistema ir logais.

## 1. Reikalavimai

- Node.js 20 arba naujesnis
- Discord bot aplikacija
- Bot token
- Serverio ID
- Bot Application ID

## 2. Įdiegimas

Atidaryk terminalą projekto aplanke:

```bash
npm install
