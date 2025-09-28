require('dotenv').config();
const { Client, GatewayIntentBits, Partials, REST, Routes, SlashCommandBuilder, PermissionFlagsBits } = require('discord.js');
const persist = require('node-persist');

const TOKEN = process.env.TOKEN;
const CLIENT_ID = process.env.CLIENT_ID;
const OWNER_ID = process.env.OWNER_ID;
const GUILD_ID = process.env.GUILD_ID || null;
const AUTO_QUARANTINE = (process.env.AUTO_QUARANTINE || 'true') === 'true';
const QUARANTINE_ROLE_NAME = process.env.QUARANTINE_ROLE_NAME || 'quarantine';
const PROTECTED_IDS = (process.env.PROTECTED_IDS || '').split(',').map(s => s.trim()).filter(Boolean).map(id => id);
const CHANNEL_DELETE_THRESHOLD = Number(process.env.CHANNEL_DELETE_THRESHOLD || 5);
const BAN_THRESHOLD = Number(process.env.BAN_THRESHOLD || 5);
const ROLE_DELETE_THRESHOLD = Number(process.env.ROLE_DELETE_THRESHOLD || 5);
const TIME_WINDOW_MS = Number(process.env.TIME_WINDOW_MS || 60_000);

if (!TOKEN || !CLIENT_ID || !OWNER_ID) {
  console.error('Please set TOKEN, CLIENT_ID and OWNER_ID in .env');
  process.exit(1);
}

// init storage
(async () => {
  await persist.init({ dir: './storage' });
})();

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.GuildMessageReactions,
    GatewayIntentBits.GuildModeration, // for ban events / audit log - note: permission needed
    GatewayIntentBits.GuildMessageTyping,
  ],
  partials: [Partials.Channel]
});

// Simple in-memory buffers to count events in sliding window
let channelDeleteTimestamps = [];
let roleDeleteTimestamps = [];
let banTimestamps = [];
let messageFloods = {}; // { guildId: { userId: [timestamps...] } }

// Register a few slash commands (backup / restore / status) for convenience
const commands = [
  new SlashCommandBuilder().setName('backup').setDescription('Take a metadata backup of this server (owner/admin only)'),
  new SlashCommandBuilder().setName('restore').setDescription('Restore channels from last backup (owner/admin only)'),
  new SlashCommandBuilder().setName('status').setDescription('Show watchdog status for this server')
].map(c => c.toJSON());

(async () => {
  const rest = new REST({ version: '10' }).setToken(TOKEN);
  try {
    if (GUILD_ID) {
      await rest.put(Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID), { body: commands });
      console.log('Registered guild commands.');
    } else {
      await rest.put(Routes.applicationCommands(CLIENT_ID), { body: commands });
      console.log('Registered global commands (may take up to an hour).');
    }
  } catch (err) {
    console.warn('Command registration failed', err);
  }
})();

client.once('ready', () => {
  console.log(`Logged in as ${client.user.tag}`);
});

// ------------------ Backup / Restore Utilities ------------------
async function backupGuildMetadata(guild) {
  const data = {
    ts: Date.now(),
    guildId: guild.id,
    channels: []
  };
  for (const ch of guild.channels.cache.values()) {
    data.channels.push({
      id: ch.id,
      name: ch.name,
      type: ch.type,
      parentId: ch.parentId,
      position: ch.position,
      permissionOverwrites: ch.permissionOverwrites.cache.map(po => ({
        id: po.id,
        allow: po.allow.bitfield.toString(),
        deny: po.deny.bitfield.toString(),
        type: po.type
      }))
    });
  }
  await persist.setItem(`backup_${guild.id}`, data);
  return data;
}

async function getBackup(guildId) {
  return await persist.getItem(`backup_${guildId}`);
}

async function restoreFromBackup(guild, sendLog = true) {
  const backup = await getBackup(guild.id);
  if (!backup) {
    if (sendLog) await ensureAlertChannel(guild).then(ch => ch.send('No backup available to restore.'));
    return { recreated: 0, missing: [] };
  }

  let recreated = 0;
  const missing = [];
  // Recreate channels that no longer exist by name
  for (const chMeta of backup.channels) {
    // find by name and type
    const exists = guild.channels.cache.find(c => c.name === chMeta.name && c.type === chMeta.type);
    if (!exists) {
      try {
        await guild.channels.create({
          name: chMeta.name,
          type: chMeta.type,
          reason: 'Restoring from watchdog backup'
        });
        recreated++;
      } catch (err) {
        console.warn('Failed to recreate', chMeta.name, err);
        missing.push(chMeta.name);
      }
      // small pause to reduce rate-limit pressure
      await new Promise(r => setTimeout(r, 350));
    }
  }

  if (sendLog) {
    const ch = await ensureAlertChannel(guild);
    await ch.send(`Restore executed. Channels recreated: ${recreated}. Missing/failed: ${missing.length}`);
  }
  return { recreated, missing };
}

// ------------------ Alert channel ------------------
async function ensureAlertChannel(guild) {
  let ch = guild.channels.cache.find(c => c.name === 'alerts' && c.isTextBased());
  if (!ch) {
    try {
      ch = await guild.channels.create({
        name: 'alerts',
        type: 0,
        reason: 'Created by watchdog to post alerts'
      });
    } catch (err) {
      console.warn('Could not create/find alerts channel', err);
      // fallback: attempt to DM owner
      ch = null;
    }
  }
  return ch;
}

async function notifyOwner(text, guild) {
  try {
    const owner = await client.users.fetch(OWNER_ID);
    await owner.send(`Watchdog alert for guild ${guild ? guild.name : '(unknown)'}:\n${text}`).catch(() => {});
  } catch (e) {
    console.warn('Failed to notify owner', e);
  }
}

// ------------------ Quarantine logic ------------------
async function ensureQuarantineRole(guild) {
  let role = guild.roles.cache.find(r => r.name === QUARANTINE_ROLE_NAME);
  if (!role) {
    try {
      role = await guild.roles.create({ name: QUARANTINE_ROLE_NAME, permissions: [], reason: 'Created as quarantine role for watchdog' });
    } catch (err) {
      console.warn('Failed to create quarantine role', err);
      return null;
    }
  }
  return role;
}

async function quarantineMember(guild, member, reason = 'Quarantined by watchdog') {
  // don't touch guild owner or protected IDs
  if (!member || member.user.bot) return { ok: false, reason: 'invalid member or bot' };
  if (member.id === guild.ownerId) return { ok: false, reason: 'guild owner' };
  if (PROTECTED_IDS.includes(member.id)) return { ok: false, reason: 'protected id' };

  const role = await ensureQuarantineRole(guild);
  if (!role) return { ok: false, reason: 'no quarantine role' };

  // Save current role IDs to storage so we can restore later
  const keptRoleIds = member.roles.cache.filter(r => r.managed === false && r.id !== guild.id).map(r => r.id); // exclude everyone role and managed roles
  await persist.setItem(`roles_backup_${guild.id}_${member.id}`, { ts: Date.now(), roles: keptRoleIds });

  // Remove all roles (except everyone and roles in PROTECTED_IDS if any have special handling)
  try {
    // Prepare new set: only quarantine role
    const newRoles = [role.id];
    await member.roles.set(newRoles, reason);
    return { ok: true, reason: 'quarantined', previousRoles: keptRoleIds };
  } catch (err) {
    console.warn('Failed to set roles for quarantine', err);
    return { ok: false, reason: 'failed to set roles' };
  }
}

async function restoreMemberRoles(guild, memberId) {
  const backup = await persist.getItem(`roles_backup_${guild.id}_${memberId}`);
  if (!backup) return { ok: false, reason: 'no backup' };
  try {
    const member = await guild.members.fetch(memberId).catch(() => null);
    if (!member) return { ok: false, reason: 'member not found' };
    await member.roles.set(backup.roles, 'Restoring roles after quarantine');
    return { ok: true };
  } catch (err) {
    console.warn('Failed to restore roles', err);
    return { ok: false, reason: 'failed to restore' };
  }
}

// ------------------ Detection / Event handlers ------------------
function pruneOldTimestamps(arr) {
  const cutoff = Date.now() - TIME_WINDOW_MS;
  return arr.filter(ts => ts > cutoff);
}

async function handlePotentialRaid(guild, reasonDescription, suspectedActorIds = []) {
  // Alert first
  const alertCh = await ensureAlertChannel(guild);
  const text = `⚠️ Potential raid detected: ${reasonDescription}\nSuspected actors: ${suspectedActorIds.join(', ') || 'unknown'}`;
  if (alertCh) await alertCh.send(text);
  await notifyOwner(text, guild);

  // Attempt to quarantine suspected actors if configured
  if (AUTO_QUARANTINE && suspectedActorIds.length) {
    for (const actorId of suspectedActorIds) {
      try {
        const member = await guild.members.fetch(actorId).catch(() => null);
        if (!member) continue;
        const res = await quarantineMember(guild, member, 'Auto-quarantine: suspected raid activity');
        if (alertCh) await alertCh.send(`Attempted quarantine on <@${actorId}>: ${res.ok ? 'success' : `failed (${res.reason})`}`);
      } catch (err) {
        console.warn('Quarantine attempt failed', err);
      }
      // small delay to avoid hammering rate limits
      await new Promise(r => setTimeout(r, 400));
    }
  }

  // Attempt to restore channels from backup
  try {
    await restoreFromBackup(guild, true);
  } catch (err) {
    console.warn('Restore attempt failed', err);
    if (alertCh) await alertCh.send('Automatic restore failed: ' + (err.message || String(err)));
  }
}

// channel deletions
client.on('channelDelete', async (channel) => {
  if (!channel.guild) return;
  channelDeleteTimestamps.push(Date.now());
  channelDeleteTimestamps = pruneOldTimestamps(channelDeleteTimestamps);

  if (channelDeleteTimestamps.length >= CHANNEL_DELETE_THRESHOLD) {
    // try to find actor via audit logs (who deleted recent channels)
    const guild = channel.guild;
    let actorIds = [];
    try {
      const logs = await guild.fetchAuditLogs({ type: 12, limit: 6 }); // 12 = CHANNEL_DELETE
      const entries = logs.entries.map(e => e[1]);
      // collect unique actor ids in recent entries
      for (const ent of entries) {
        if (Date.now() - ent.createdTimestamp < TIME_WINDOW_MS * 2) actorIds.push(ent.executor.id);
      }
      actorIds = Array.from(new Set(actorIds)).filter(id => id !== client.user.id);
    } catch (err) {
      console.warn('Audit log fetch failed', err);
    }

    await handlePotentialRaid(channel.guild, `${channelDeleteTimestamps.length} channels deleted within ${TIME_WINDOW_MS}ms`, actorIds);
    channelDeleteTimestamps = [];
  }
});

// role deletions
client.on('roleDelete', async (role) => {
  if (!role.guild) return;
  roleDeleteTimestamps.push(Date.now());
  roleDeleteTimestamps = pruneOldTimestamps(roleDeleteTimestamps);

  if (roleDeleteTimestamps.length >= ROLE_DELETE_THRESHOLD) {
    const guild = role.guild;
    let actorIds = [];
    try {
      const logs = await guild.fetchAuditLogs({ type: 32, limit: 6 }); // 32 = ROLE_DELETE
      for (const e of logs.entries.map(p => p[1])) {
        if (Date.now() - e.createdTimestamp < TIME_WINDOW_MS * 2) actorIds.push(e.executor.id);
      }
      actorIds = Array.from(new Set(actorIds)).filter(id => id !== client.user.id);
    } catch (err) {
      console.warn('Audit log fetch failed', err);
    }

    await handlePotentialRaid(role.guild, `${roleDeleteTimestamps.length} roles deleted within ${TIME_WINDOW_MS}ms`, actorIds);
    roleDeleteTimestamps = [];
  }
});

// bans
client.on('guildBanAdd', async (ban) => {
  if (!ban.guild) return;
  banTimestamps.push(Date.now());
  banTimestamps = pruneOldTimestamps(banTimestamps);

  if (banTimestamps.length >= BAN_THRESHOLD) {
    const guild = ban.guild;
    let actorIds = [];
    try {
      const logs = await guild.fetchAuditLogs({ type: 22, limit: 6 }); // 22 = MEMBER_BAN_ADD
      for (const e of logs.entries.map(p => p[1])) {
        if (Date.now() - e.createdTimestamp < TIME_WINDOW_MS * 2) actorIds.push(e.executor.id);
      }
      actorIds = Array.from(new Set(actorIds)).filter(id => id !== client.user.id);
    } catch (err) {
      console.warn('Audit log fetch failed', err);
    }
    await handlePotentialRaid(ban.guild, `${banTimestamps.length} bans within ${TIME_WINDOW_MS}ms`, actorIds);
    banTimestamps = [];
  }
});

// message flood detection (simple per-user flood across channels)
client.on('messageCreate', async (message) => {
  if (!message.guild || message.author.bot) return;
  const gid = message.guild.id;
  messageFloods[gid] = messageFloods[gid] || {};
  const arr = messageFloods[gid][message.author.id] = (messageFloods[gid][message.author.id] || []);
  arr.push(Date.now());
  // prune
  messageFloods[gid][message.author.id] = arr.filter(ts => ts > Date.now() - TIME_WINDOW_MS);
  if (messageFloods[gid][message.author.id].length >= 25) { // 25 messages in TIME_WINDOW => suspicious
    await handlePotentialRaid(message.guild, `Message flood: ${message.author.tag} sent ${messageFloods[gid][message.author.id].length} messages in ${TIME_WINDOW_MS}ms`, [message.author.id]);
    messageFloods[gid][message.author.id] = [];
  }
});

// ------------------ Interaction handlers (slash commands) ------------------
client.on('interactionCreate', async (interaction) => {
  if (!interaction.isChatInputCommand()) return;
  const cmd = interaction.commandName;
  const member = interaction.guild ? await interaction.guild.members.fetch(interaction.user.id).catch(() => null) : null;
  const isOwner = interaction.user.id === OWNER_ID;
  const isAdmin = member ? member.permissions.has(PermissionFlagsBits.Administrator) : false;

  if (cmd === 'backup') {
    if (!isOwner && !isAdmin) return interaction.reply({ content: 'Admin or owner only.', ephemeral: true });
    await interaction.deferReply({ ephemeral: true });
    const data = await backupGuildMetadata(interaction.guild);
    await interaction.editReply({ content: `Backup saved. Channels backed up: ${data.channels.length}`, ephemeral: true });
    return;
  }

  if (cmd === 'restore') {
    if (!isOwner && !isAdmin) return interaction.reply({ content: 'Admin or owner only.', ephemeral: true });
    await interaction.deferReply({ ephemeral: true });
    const result = await restoreFromBackup(interaction.guild, true);
    await interaction.editReply({ content: `Restore complete. Channels recreated: ${result.recreated}`, ephemeral: true });
    return;
  }

  if (cmd === 'status') {
    // show simple metrics
    const txt = `Watchdog Status\nChannel deletes in window: ${pruneOldTimestamps(channelDeleteTimestamps).length}\nRole deletes in window: ${pruneOldTimestamps(roleDeleteTimestamps).length}\nRecent bans: ${pruneOldTimestamps(banTimestamps).length}\nAuto-quarantine: ${AUTO_QUARANTINE}\nQuarantine role name: ${QUARANTINE_ROLE_NAME}`;
    await interaction.reply({ content: txt, ephemeral: true });
    return;
  }
});

// Periodic metadata backup for all guilds every 5 minutes
setInterval(async () => {
  for (const g of client.guilds.cache.values()) {
    try {
      await backupGuildMetadata(g);
    } catch (err) {
      console.warn('Periodic backup error for guild', g.id, err);
    }
  }
}, 5 * 60 * 1000);

client.login(TOKEN);