
/**
 * VK-прокси сервер для отправки уведомлений о новых заявках.
 *
 * Запускается отдельно от основного сайта (на Render / Railway / любой Node.js хостинг).
 * Принимает POST с leadId, читает заявку из БД Lork и отправляет сообщение в VK.
 *
 * Переменные окружения (обязательные):
 *   VK_GROUP_TOKEN      — токен сообщества VK (с правом messages)
 *   VK_NOTIFY_PEER_ID   — ID пользователя или беседы для уведомлений
 *   VK_API_VERSION      — версия VK API (5.199)
 *   VITE_LORK_BAAS_URL  — URL BaaS Lork
 *   VITE_LORK_PROJECT_ID — ID проекта Lork
 *   VITE_LORK_ANON_KEY  — анонимный ключ Lork
 *   PORT                — порт (по умолчанию 3001)
 */

import http from 'http';
import { URL } from 'url';

// ============================================================
// Загрузка переменных окружения
// ============================================================
const REQUIRED_ENV = [
  'VK_GROUP_TOKEN',
  'VK_NOTIFY_PEER_ID',
  'VITE_LORK_BAAS_URL',
  'VITE_LORK_PROJECT_ID',
  'VITE_LORK_ANON_KEY',
];

function checkEnv() {
  const missing = REQUIRED_ENV.filter((key) => !process.env[key]);
  if (missing.length > 0) {
    console.error(`[VK-SERVER] Ошибка: отсутствуют переменные окружения: ${missing.join(', ')}`);
    console.error('[VK-SERVER] Сервер не может работать без них.');
    return false;
  }
  return true;
}

// ============================================================
// Отправка сообщения в VK
// ============================================================
async function sendVkMessage(message) {
  const token = process.env.VK_GROUP_TOKEN;
  const peerId = process.env.VK_NOTIFY_PEER_ID;
  const apiVersion = process.env.VK_API_VERSION || '5.199';

  const params = new URLSearchParams({
    access_token: token,
    v: apiVersion,
    peer_id: peerId,
    random_id: String(Date.now()),
    message: message,
  });

  const response = await fetch('https://api.vk.com/method/messages.send', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params.toString(),
  });

  const result = await response.json();

  if (result.error) {
    return { success: false, error: result.error };
  }

  return { success: true, error: null };
}

// ============================================================
// Чтение заявки из БД Lork
// ============================================================
async function fetchLeadFromDb(leadId) {
  const baasUrl = process.env.VITE_LORK_BAAS_URL;
  const projectId = process.env.VITE_LORK_PROJECT_ID;
  const anonKey = process.env.VITE_LORK_ANON_KEY;

  const url = `${baasUrl}/${projectId}/db/leads?id=eq.${leadId}&select=*`;

  const response = await fetch(url, {
    headers: {
      Authorization: `Bearer ${anonKey}`,
      'Accept-Profile': projectId,
      'Content-Profile': projectId,
    },
  });

  if (!response.ok) {
    throw new Error(`DB request failed: ${response.status} ${response.statusText}`);
  }

  const data = await response.json();
  return data?.[0] || null;
}

// ============================================================
// Обновление статуса VK в БД
// ============================================================
async function updateVkStatusInDb(leadId, vkSent, vkError) {
  const baasUrl = process.env.VITE_LORK_BAAS_URL;
  const projectId = process.env.VITE_LORK_PROJECT_ID;
  const anonKey = process.env.VITE_LORK_ANON_KEY;

  const url = `${baasUrl}/${projectId}/db/leads?id=eq.${leadId}`;

  const body = {
    vk_sent: vkSent,
    vk_error: vkError || null,
  };

  const response = await fetch(url, {
    method: 'PATCH',
    headers: {
      Authorization: `Bearer ${anonKey}`,
      'Accept-Profile': projectId,
      'Content-Profile': projectId,
      'Content-Type': 'application/json',
      Prefer: 'return=minimal',
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    console.error(`[VK-SERVER] Не удалось обновить статус VK для заявки ${leadId}: ${response.status}`);
  }
}

// ============================================================
// Форматирование сообщения для VK
// ============================================================
function formatVkMessage(lead) {
  const lines = [
    '━━━━━━━━━━━━━━━━━━━━',
    '🚀 НОВАЯ ЗАЯВКА НА ОТКРЫТЫЙ УРОК',
    '━━━━━━━━━━━━━━━━━━━━',
    '',
    `👤 Имя: ${lead.name || 'не указано'}`,
    `📞 Телефон: ${lead.phone || 'не указан'}`,
    `👥 Аудитория: ${lead.audience || 'не указана'}`,
  ];

  if (lead.age) {
    lines.push(`🎂 Возраст: ${lead.age}`);
  }

  lines.push(`📌 Направление: ${lead.direction || 'не указано'}`);

  if (lead.comment) {
    lines.push(`💬 Комментарий: ${lead.comment}`);
  }

  lines.push('');
  lines.push(`🆔 ID заявки: ${lead.id}`);
  lines.push(`📅 Дата: ${new Date(lead.created_at).toLocaleString('ru-RU')}`);
  lines.push(`🌐 Источник: ${lead.page_url || 'сайт'}`);
  lines.push('');
  lines.push('━━━━━━━━━━━━━━━━━━━━');

  return lines.join('\n');
}

// ============================================================
// HTTP сервер
// ============================================================
const PORT = process.env.PORT || 3001;

const server = http.createServer(async (req, res) => {
  // CORS
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }

  // Только POST /api/vk-notify
  if (req.method !== 'POST' || req.url !== '/api/vk-notify') {
    res.writeHead(404, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Not found' }));
    return;
  }

  // Читаем тело запроса
  let body = '';
  req.on('data', (chunk) => (body += chunk));
  req.on('end', async () => {
    let leadId;
    try {
      const parsed = JSON.parse(body);
      leadId = parsed.leadId;
    } catch {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Invalid JSON' }));
      return;
    }

    if (!leadId) {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'leadId is required' }));
      return;
    }

    // Проверяем переменные окружения
    if (!checkEnv()) {
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: false,
        error: 'Server configuration error: missing environment variables',
      }));
      return;
    }

    try {
      // Читаем заявку из БД
      const lead = await fetchLeadFromDb(leadId);

      if (!lead) {
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Lead not found' }));
        return;
      }

      // Форматируем и отправляем в VK
      const message = formatVkMessage(lead);
      const vkResult = await sendVkMessage(message);

      // Обновляем статус в БД
      await updateVkStatusInDb(leadId, vkResult.success, vkResult.error?.error_msg || null);

      // Отвечаем
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: true,
        leadId: lead.id,
        vkSent: vkResult.success,
        vkError: vkResult.error?.error_msg || null,
      }));

      console.log(`[VK-SERVER] Заявка ${leadId}: VK ${vkResult.success ? 'отправлено' : 'ошибка: ' + (vkResult.error?.error_msg || 'неизвестная')}`);
    } catch (err) {
      console.error(`[VK-SERVER] Ошибка обработки заявки ${leadId}:`, err);
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: false,
        error: err.message || 'Internal server error',
      }));
    }
  });
});

server.listen(PORT, () => {
  console.log(`[VK-SERVER] Запущен на порту ${PORT}`);
  console.log(`[VK-SERVER] Endpoint: POST /api/vk-notify`);

  if (checkEnv()) {
    console.log(`[VK-SERVER] VK_NOTIFY_PEER_ID: ${process.env.VK_NOTIFY_PEER_ID}`);
    console.log(`[VK-SERVER] VK_API_VERSION: ${process.env.VK_API_VERSION || '5.199'}`);
    console.log(`[VK-SERVER] VK_GROUP_TOKEN: задан (${process.env.VK_GROUP_TOKEN.length} символов)`);
  }
});
