const APP = Object.freeze({
  title: '家長通知中心',
  timezone: 'Asia/Taipei',
  dataSheetName: '通知資料',
  settingsSheetName: '系統設定',
  imageFolderName: '家長通知圖片',
  categories: ['收費通知', '課程通知', '報名通知', '圖片通知'],
  headers: [
    'ID', '標題', '通知日期', '月份', '分類', '通知內容',
    '圖片檔案ID', '圖片網址', '已發布', '置頂',
    '開始顯示日', '結束顯示日', '建立時間', '更新時間'
  ],
  col: Object.freeze({
    id: 1,
    title: 2,
    noticeDate: 3,
    month: 4,
    category: 5,
    content: 6,
    imageFileId: 7,
    imageUrl: 8,
    published: 9,
    pinned: 10,
    startDate: 11,
    endDate: 12,
    createdAt: 13,
    updatedAt: 14
  })
});

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('家長通知系統')
    .addItem('初始化系統', 'setupSystem')
    .addItem('開啟管理頁', 'showAdminSidebar')
    .addSeparator()
    .addItem('重新整理管理頁', 'showAdminSidebar')
    .addToUi();
}

function doGet(e) {
  const requestedView = e && e.parameter && e.parameter.view === 'admin'
    ? 'admin'
    : 'public';
  return renderPage_(requestedView);
}

function doPost(e) {
  try {
    if (!e || !e.postData || !e.postData.contents) {
      return ContentService.createTextOutput('No content').setMimeType(ContentService.MimeType.TEXT);
    }
    const data = JSON.parse(e.postData.contents);
    const events = data.events || [];
    for (let i = 0; i < events.length; i++) {
      handleLineEvent_(events[i]);
    }
    return ContentService.createTextOutput(JSON.stringify({ status: 'ok' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    console.error('doPost error:', err);
    return ContentService.createTextOutput(JSON.stringify({ status: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function showAdminSidebar() {
  const template = HtmlService.createTemplateFromFile('Index');
  template.initialView = 'admin';
  SpreadsheetApp.getUi().showSidebar(
    template.evaluate().setTitle(APP.title + ' 管理')
  );
}

function setupSystem() {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  if (!spreadsheet) {
    throw new Error('請從綁定通知試算表的 Apps Script 執行初始化。');
  }

  spreadsheet.setSpreadsheetTimeZone(APP.timezone);
  const dataSheet = ensureDataSheet_(spreadsheet);
  ensureSettingsSheet_(spreadsheet);

  const adminEmail = Session.getActiveUser().getEmail();
  if (!adminEmail) {
    throw new Error('無法取得管理者 Google 帳號，請以自己的帳號登入後再初始化。');
  }

  const properties = PropertiesService.getScriptProperties();
  if (!properties.getProperty('ADMIN_EMAIL')) {
    properties.setProperty('ADMIN_EMAIL', adminEmail.toLowerCase());
  }
  properties.setProperty('SPREADSHEET_ID', spreadsheet.getId());

  let folderId = properties.getProperty('IMAGE_FOLDER_ID');
  if (!folderId) {
    const folder = DriveApp.createFolder(spreadsheet.getName() + ' - ' + APP.imageFolderName);
    folderId = folder.getId();
    properties.setProperty('IMAGE_FOLDER_ID', folderId);
  }

  writeSettings_(spreadsheet, {
    ADMIN_EMAIL: properties.getProperty('ADMIN_EMAIL'),
    IMAGE_FOLDER_ID: folderId,
    SPREADSHEET_ID: spreadsheet.getId(),
    IMAGE_FOLDER_URL: 'https://drive.google.com/drive/folders/' + folderId
  });

  dataSheet.activate();
  return {
    adminEmail: properties.getProperty('ADMIN_EMAIL'),
    imageFolderId: folderId,
    spreadsheetUrl: spreadsheet.getUrl(),
    message: '系統初始化完成。'
  };
}

function getPublicBootstrap() {
  const notices = readNotices_(true).map(function(notice) {
    return {
      title: notice.title,
      noticeDate: notice.noticeDate,
      month: notice.month,
      category: notice.category,
      content: notice.content,
      imageUrl: notice.imageUrl,
      pinned: notice.pinned
    };
  });
  const months = unique_(notices.map(function(notice) { return notice.month; }))
    .sort()
    .reverse();
  return {
    categories: APP.categories,
    months: months,
    notices: notices
  };
}

function getAdminBootstrap() {
  assertAdmin_();
  const spreadsheet = getSpreadsheet_();
  return {
    categories: APP.categories,
    notices: readNotices_(false),
    adminEmail: Session.getActiveUser().getEmail(),
    spreadsheetUrl: spreadsheet.getUrl(),
    imageFolderId: PropertiesService.getScriptProperties().getProperty('IMAGE_FOLDER_ID') || ''
  };
}

function saveNotice(payload) {
  assertAdmin_();
  return saveNoticeInternal_(payload);
}

function saveNoticeInternal_(payload) {
  const notice = validateNoticePayload_(payload || {});
  const sheet = getDataSheet_();
  if (!sheet) {
    throw new Error('找不到通知資料工作表，請先從試算表選單執行「初始化系統」。');
  }

  const now = new Date();
  const existingRow = notice.id ? findRowById_(sheet, notice.id) : 0;
  let createdAt = now;
  if (existingRow) {
    const existing = sheet.getRange(existingRow, 1, 1, APP.headers.length).getValues()[0];
    createdAt = existing[APP.col.createdAt - 1] || now;
  }

  const row = [
    notice.id || Utilities.getUuid(),
    notice.title,
    parseDateInput_(notice.noticeDate),
    notice.noticeDate.slice(0, 7),
    notice.category,
    notice.content,
    notice.imageFileId,
    notice.imageUrl,
    notice.published,
    notice.pinned,
    notice.startDate ? parseDateInput_(notice.startDate) : '',
    notice.endDate ? parseDateInput_(notice.endDate) : '',
    createdAt,
    now
  ];

  const rowNumber = existingRow || Math.max(sheet.getLastRow() + 1, 2);
  sheet.getRange(rowNumber, 1, 1, APP.headers.length).setValues([row]);
  formatDataRows_(sheet, rowNumber, 1);
  return rowToNotice_(row, rowNumber);
}

function uploadImage(dataUrl, fileName, mimeType) {
  assertAdmin_();
  if (!dataUrl || String(dataUrl).indexOf('base64,') === -1) {
    throw new Error('圖片資料格式不正確。');
  }

  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
  const safeMimeType = String(mimeType || '').toLowerCase();
  if (allowedTypes.indexOf(safeMimeType) === -1) {
    throw new Error('圖片格式只支援 JPG、PNG、GIF 或 WebP。');
  }

  const encoded = String(dataUrl).split('base64,')[1];
  const bytes = Utilities.base64Decode(encoded);
  if (bytes.length > 8 * 1024 * 1024) {
    throw new Error('圖片大小不可超過 8 MB。');
  }

  const folderId = PropertiesService.getScriptProperties().getProperty('IMAGE_FOLDER_ID');
  if (!folderId) {
    throw new Error('尚未設定圖片資料夾，請先初始化系統。');
  }

  const folder = DriveApp.getFolderById(folderId);
  const safeName = sanitizeFileName_(fileName || '通知圖片') || '通知圖片';
  const file = folder.createFile(Utilities.newBlob(bytes, safeMimeType, safeName));
  let sharingWarning = '';
  try {
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
  } catch (error) {
    sharingWarning = '圖片已上傳，但目前 Google Workspace 政策不允許公開連結，請確認 Drive 共用權限。';
  }

  return {
    fileId: file.getId(),
    fileName: file.getName(),
    imageUrl: driveImageUrl_(file.getId()),
    sharingWarning: sharingWarning
  };
}

function onEdit(e) {
  if (!e || !e.range) return;
  const sheet = e.range.getSheet();
  if (sheet.getName() !== APP.dataSheetName || e.range.getRow() < 2) return;

  const firstRow = e.range.getRow();
  const rowCount = e.range.getNumRows();
  const rows = sheet.getRange(firstRow, 1, rowCount, APP.headers.length).getValues();
  const dateValues = rows.map(function(row) { return [row[APP.col.noticeDate - 1]]; });
  const ids = rows.map(function(row) {
    const currentId = String(row[APP.col.id - 1] || '');
    const hasData = [
      row[APP.col.title - 1],
      row[APP.col.noticeDate - 1],
      row[APP.col.category - 1],
      row[APP.col.content - 1],
      row[APP.col.imageFileId - 1],
      row[APP.col.imageUrl - 1]
    ].some(function(value) { return value !== '' && value != null; });
    return [currentId || (hasData ? Utilities.getUuid() : '')];
  });
  const months = dateValues.map(function(row) {
    const date = parseDateInput_(row[0]);
    return [date ? formatMonth_(date) : ''];
  });
  const now = new Date();
  const timestamps = dateValues.map(function() { return [now]; });
  sheet.getRange(firstRow, APP.col.id, rowCount, 1).setValues(ids);
  sheet.getRange(firstRow, APP.col.month, rowCount, 1).setValues(months);
  sheet.getRange(firstRow, APP.col.updatedAt, rowCount, 1).setValues(timestamps);
  formatDataRows_(sheet, firstRow, rowCount);
}

function renderPage_(view) {
  const template = HtmlService.createTemplateFromFile('Index');
  template.initialView = view;
  return template.evaluate()
    .setTitle(APP.title)
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function ensureDataSheet_(spreadsheet) {
  let sheet = spreadsheet.getSheetByName(APP.dataSheetName);
  if (!sheet) sheet = spreadsheet.insertSheet(APP.dataSheetName);

  const headerValues = sheet.getRange(1, 1, 1, APP.headers.length).getValues()[0];
  const isEmpty = headerValues.every(function(value) { return value === ''; });
  if (isEmpty) {
    sheet.getRange(1, 1, 1, APP.headers.length).setValues([APP.headers]);
  } else if (headerValues.slice(0, APP.headers.length).join('|') !== APP.headers.join('|')) {
    throw new Error('通知資料的欄位標題與系統預期不同，未自動覆蓋既有資料。');
  }

  sheet.setFrozenRows(1);
  sheet.getRange(1, 1, 1, APP.headers.length)
    .setFontWeight('bold')
    .setFontColor('#ffffff')
    .setBackground('#173f5f');
  const validationRows = Math.max(sheet.getMaxRows() - 1, 1);
  sheet.getRange(2, APP.col.category, validationRows, 1).setDataValidation(
    SpreadsheetApp.newDataValidation()
      .requireValueInList(APP.categories, true)
      .setAllowInvalid(false)
      .build()
  );
  sheet.getRange(2, APP.col.published, validationRows, 2).setDataValidation(
    SpreadsheetApp.newDataValidation().requireCheckbox().build()
  );
  formatDataRows_(sheet, 2, Math.max(sheet.getMaxRows() - 1, 1));
  sheet.autoResizeColumns(1, APP.headers.length);
  return sheet;
}

function ensureSettingsSheet_(spreadsheet) {
  let sheet = spreadsheet.getSheetByName(APP.settingsSheetName);
  if (!sheet) sheet = spreadsheet.insertSheet(APP.settingsSheetName);
  if (sheet.getRange('A1').getValue() === '') {
    sheet.getRange('A1:B1').setValues([['設定項目', '值']]);
    sheet.getRange('A1:B1').setFontWeight('bold').setBackground('#e8eef4');
  }
  return sheet;
}

function writeSettings_(spreadsheet, settings) {
  const sheet = ensureSettingsSheet_(spreadsheet);
  const rows = Object.keys(settings).map(function(key) { return [key, settings[key]]; });
  if (rows.length) sheet.getRange(2, 1, rows.length, 2).setValues(rows);
  sheet.autoResizeColumns(1, 2);
}

function readNotices_(publicOnly) {
  const sheet = getDataSheet_();
  if (!sheet || sheet.getLastRow() < 2) return [];

  const values = sheet.getRange(2, 1, sheet.getLastRow() - 1, APP.headers.length).getValues();
  const today = formatDate_(new Date());
  return values
    .map(function(row, index) { return rowToNotice_(row, index + 2); })
    .filter(function(notice) {
      if (!notice.id || !notice.title || !notice.noticeDate) return false;
      if (!publicOnly) return true;
      return notice.published &&
        (!notice.startDate || notice.startDate <= today) &&
        (!notice.endDate || notice.endDate >= today);
    })
    .sort(sortNotices_);
}

function rowToNotice_(row, rowNumber) {
  const noticeDate = toDateInput_(row[APP.col.noticeDate - 1]);
  const month = noticeDate ? noticeDate.slice(0, 7) : String(row[APP.col.month - 1] || '');
  const storedTitle = String(row[APP.col.title - 1] || '').trim();
  const content = String(row[APP.col.content - 1] || '').trim();
  const category = String(row[APP.col.category - 1] || '');
  const title = storedTitle || (category === '圖片通知' ? content.split(/\r?\n/)[0].slice(0, 100) : '');
  const storedImageUrl = String(row[APP.col.imageUrl - 1] || '');
  const storedImageFileId = String(row[APP.col.imageFileId - 1] || '');
  const imageFileId = storedImageFileId || extractDriveFileId_(storedImageUrl);
  return {
    rowNumber: rowNumber,
    id: String(row[APP.col.id - 1] || ''),
    title: title,
    noticeDate: noticeDate,
    month: month,
    category: category,
    content: content,
    imageFileId: imageFileId,
    imageUrl: normalizeImageUrl_(storedImageUrl, imageFileId),
    published: toBoolean_(row[APP.col.published - 1]),
    pinned: toBoolean_(row[APP.col.pinned - 1]),
    startDate: toDateInput_(row[APP.col.startDate - 1]),
    endDate: toDateInput_(row[APP.col.endDate - 1]),
    createdAt: toTimestampText_(row[APP.col.createdAt - 1]),
    updatedAt: toTimestampText_(row[APP.col.updatedAt - 1])
  };
}

function validateNoticePayload_(payload) {
  const title = cleanText_(payload.title, 200);
  const noticeDate = cleanText_(payload.noticeDate, 10);
  const category = cleanText_(payload.category, 30);
  const content = cleanText_(payload.content, 10000);
  const startDate = cleanText_(payload.startDate, 10);
  const endDate = cleanText_(payload.endDate, 10);

  if (!title) throw new Error('請填寫通知標題。');
  if (!/^\d{4}-\d{2}-\d{2}$/.test(noticeDate) || !parseDateInput_(noticeDate)) {
    throw new Error('通知日期格式不正確。');
  }
  if (APP.categories.indexOf(category) === -1) throw new Error('通知分類不正確。');
  if (startDate && !/^\d{4}-\d{2}-\d{2}$/.test(startDate)) throw new Error('開始顯示日格式不正確。');
  if (endDate && !/^\d{4}-\d{2}-\d{2}$/.test(endDate)) throw new Error('結束顯示日格式不正確。');
  if (startDate && endDate && startDate > endDate) throw new Error('開始顯示日不可晚於結束顯示日。');
  if (!content && !payload.imageUrl) throw new Error('請填寫通知內容或上傳圖片。');

  return {
    id: cleanText_(payload.id, 80),
    title: title,
    noticeDate: noticeDate,
    category: category,
    content: content,
    imageFileId: cleanText_(payload.imageFileId, 200),
    imageUrl: cleanText_(payload.imageUrl, 500),
    published: toBoolean_(payload.published),
    pinned: toBoolean_(payload.pinned),
    startDate: startDate,
    endDate: endDate
  };
}

function assertAdmin_() {
  const expected = PropertiesService.getScriptProperties().getProperty('ADMIN_EMAIL');
  const active = Session.getActiveUser().getEmail();
  if (!expected || !active || expected.toLowerCase() !== active.toLowerCase()) {
    throw new Error('只有系統設定的管理者 Google 帳號可以操作管理功能。請從通知試算表開啟管理頁。');
  }
}

function getSpreadsheet_() {
  const props = PropertiesService.getScriptProperties();
  let id = props.getProperty('SPREADSHEET_ID');
  if (id) {
    try {
      return SpreadsheetApp.openById(id);
    } catch (e) {
      console.warn('openById failed with id:', id, e);
    }
  }
  const active = SpreadsheetApp.getActiveSpreadsheet();
  if (active) {
    props.setProperty('SPREADSHEET_ID', active.getId());
    return active;
  }
  throw new Error('尚未找到通知試算表。請至試算表重新整理頁面，點選上方功能表「家長通知系統 ➔ 初始化系統」。');
}

function getDataSheet_() {
  const ss = getSpreadsheet_();
  let sheet = ss.getSheetByName(APP.dataSheetName);
  if (!sheet) {
    sheet = ensureDataSheet_(ss);
  }
  return sheet;
}

function checkSpreadsheetSync() {
  const ss = getSpreadsheet_();
  Logger.log('=============================');
  Logger.log('📌 試算表名稱：' + ss.getName());
  Logger.log('🔗 試算表網址：' + ss.getUrl());
  const sheet = getDataSheet_();
  Logger.log('📑 資料分頁名稱：' + sheet.getName());
  Logger.log('📊 目前總列數：' + sheet.getLastRow());
  if (sheet.getLastRow() >= 2) {
    const lastRow = sheet.getRange(sheet.getLastRow(), 1, 1, APP.headers.length).getValues()[0];
    Logger.log('🟢 最新一筆通知標題：' + lastRow[APP.col.title - 1]);
    Logger.log('🟢 最新一筆通知內容：' + lastRow[APP.col.content - 1]);
  } else {
    Logger.log('⚠️ 目前分頁中尚無任何資料列。');
  }
  Logger.log('=============================');
}

function findRowById_(sheet, id) {
  if (sheet.getLastRow() < 2) return 0;
  const ids = sheet.getRange(2, APP.col.id, sheet.getLastRow() - 1, 1).getValues();
  for (let index = 0; index < ids.length; index += 1) {
    if (String(ids[index][0]) === String(id)) return index + 2;
  }
  return 0;
}

function formatDataRows_(sheet, firstRow, rowCount) {
  if (rowCount < 1) return;
  sheet.getRange(firstRow, APP.col.noticeDate, rowCount, 1).setNumberFormat('yyyy-mm-dd');
  sheet.getRange(firstRow, APP.col.startDate, rowCount, 2).setNumberFormat('yyyy-mm-dd');
  sheet.getRange(firstRow, APP.col.createdAt, rowCount, 2).setNumberFormat('yyyy-mm-dd hh:mm:ss');
}

function parseDateInput_(value) {
  if (!value) return null;
  if (Object.prototype.toString.call(value) === '[object Date]') {
    return isNaN(value.getTime()) ? null : value;
  }
  const text = String(value).trim();
  if (!text) return null;
  if (/^\d{4}-\d{2}-\d{2}$/.test(text)) {
    return Utilities.parseDate(text, APP.timezone, 'yyyy-MM-dd');
  }
  const parsed = new Date(text);
  return isNaN(parsed.getTime()) ? null : parsed;
}

function toDateInput_(value) {
  if (!value) return '';
  const text = String(value);
  if (/^\d{4}-\d{2}-\d{2}/.test(text)) return text.slice(0, 10);
  const date = parseDateInput_(value);
  return date ? formatDate_(date) : '';
}

function formatDate_(date) {
  return Utilities.formatDate(date, APP.timezone, 'yyyy-MM-dd');
}

function formatMonth_(date) {
  return Utilities.formatDate(date, APP.timezone, 'yyyy-MM');
}

function toTimestampText_(value) {
  if (!value) return '';
  if (Object.prototype.toString.call(value) === '[object Date]' && !isNaN(value.getTime())) {
    return Utilities.formatDate(value, APP.timezone, "yyyy-MM-dd'T'HH:mm:ss");
  }
  return String(value);
}

function toBoolean_(value) {
  if (value === true || value === 1) return true;
  const normalized = String(value || '').toLowerCase().trim();
  return ['true', '1', 'yes', 'y', '是', '已發布', '置頂'].indexOf(normalized) !== -1;
}

function cleanText_(value, maxLength) {
  return String(value == null ? '' : value).trim().slice(0, maxLength);
}

function driveImageUrl_(fileId) {
  return 'https://drive.google.com/thumbnail?id=' + encodeURIComponent(fileId) + '&sz=w1600';
}

function extractDriveFileId_(url) {
  const text = String(url || '').trim();
  const match = text.match(/\/file\/d\/([a-zA-Z0-9_-]+)/) ||
    text.match(/[?&]id=([a-zA-Z0-9_-]+)/);
  return match ? match[1] : '';
}

function normalizeImageUrl_(url, fileId) {
  const text = String(url || '').trim();
  return fileId ? driveImageUrl_(fileId) : text;
}

function sanitizeFileName_(fileName) {
  return String(fileName)
    .replace(/[\\/:*?"<>|]/g, '-')
    .replace(/\s+/g, ' ')
    .trim()
    .slice(0, 120);
}

function unique_(values) {
  return values.filter(function(value, index, array) {
    return value && array.indexOf(value) === index;
  });
}

function sortNotices_(left, right) {
  if (left.pinned !== right.pinned) return left.pinned ? -1 : 1;
  if (left.noticeDate !== right.noticeDate) return right.noticeDate.localeCompare(left.noticeDate);
  return right.updatedAt.localeCompare(left.updatedAt);
}

// ==========================================
// LINE Messaging API 整合模組
// ==========================================

function testLineConnection() {
  const token = PropertiesService.getScriptProperties().getProperty('LINE_CHANNEL_ACCESS_TOKEN');
  if (!token) {
    Logger.log('❌ 尚未在「專案設定 ➔ 指令碼屬性」設定 LINE_CHANNEL_ACCESS_TOKEN');
    return;
  }
  const usage = getLineMonthlyUsage_(token);
  Logger.log('✅ 恭喜！LINE API 連線測試成功！本月已使用訊息數：' + usage.totalUsage + ' 則。');
}

function handleLineEvent_(event) {
  if (!event || event.type !== 'message') return;

  const userId = event.source && event.source.userId;
  if (!userId) return;

  const props = PropertiesService.getScriptProperties();
  const token = props.getProperty('LINE_CHANNEL_ACCESS_TOKEN');
  if (!token) {
    console.warn('尚未設定 LINE_CHANNEL_ACCESS_TOKEN');
    return;
  }

  let adminUserId = props.getProperty('LINE_ADMIN_USER_ID');
  const messageType = event.message && event.message.type;

  // 1. 處理純文字訊息指令
  if (messageType === 'text') {
    const text = String(event.message.text || '').trim();

    // 指令：查詢 ID
    if (text === '#查詢ID' || text === '#myid' || text.toLowerCase() === '#id') {
      replyLineMessage_(event.replyToken, '您的 LINE User ID 為：\n' + userId + '\n\n若您是管理員，請傳送「#綁定管理員」完成系統設定。', token);
      return;
    }

    // 指令：綁定管理員
    if (text === '#綁定管理員') {
      if (!adminUserId) {
        props.setProperty('LINE_ADMIN_USER_ID', userId);
        replyLineMessage_(event.replyToken, '✅ 恭喜！您已成功綁定為家長通知中心的 LINE 管理員！\n\n您現在可以直接傳送：\n#公告 標題\n公告內容\n系統將自動登錄至雲端公告並群發給全體家長。', token);
      } else if (adminUserId === userId) {
        replyLineMessage_(event.replyToken, '您已經是此系統的管理者囉！直接傳送「#公告 標題」即可發布通知。', token);
      } else {
        replyLineMessage_(event.replyToken, '❌ 系統已綁定其他管理員帳號。若需重設，請至 Google Apps Script 的「專案設定 ➔ 指令碼屬性」更新 LINE_ADMIN_USER_ID。', token);
      }
      return;
    }

    // 身分驗證：非管理員不能使用公告發布與額度功能
    if (userId !== adminUserId) {
      if (text.indexOf('#公告') === 0 || text === '#額度') {
        replyLineMessage_(event.replyToken, '⚠️ 只有系統設定的管理員 LINE 帳號可以操作公告與查詢功能。', token);
      }
      return;
    }

    // 指令：說明手冊
    if (text === '#說明' || text === '#help') {
      replyLineMessage_(event.replyToken, getLineHelpText_(), token);
      return;
    }

    // 指令：查詢額度
    if (text === '#額度' || text === '#quota') {
      const usage = getLineMonthlyUsage_(token);
      const remaining = Math.max(0, 200 - usage.totalUsage);
      replyLineMessage_(event.replyToken, '📊【本月 LINE 訊息用量統計】\n已使用：' + usage.totalUsage + ' 則\n免費上限：200 則\n剩餘可用：' + remaining + ' 則' + (usage.totalUsage >= 180 ? '\n\n⚠️ 提醒：免費額度即將用罄！' : ''), token);
      return;
    }

    // 指令：發布公告
    if (text.indexOf('#公告') === 0) {
      processTextNoticeFromLine_(text, event.replyToken, token);
      return;
    }
  }

  // 2. 處理圖片訊息（管理員傳送圖片自動轉為圖片公告）
  if (messageType === 'image') {
    if (userId !== adminUserId) return;
    processImageNoticeFromLine_(event.message.id, event.replyToken, token);
  }
}

function processTextNoticeFromLine_(text, replyToken, token) {
  try {
    const raw = text.replace(/^#公告\s*/, '').trim();
    if (!raw) {
      replyLineMessage_(replyToken, '格式錯誤！請依照以下格式發布：\n#公告 標題\n公告內文', token);
      return;
    }

    const parsed = parseNoticeText_(raw);
    let title = parsed.title;
    const content = parsed.content;

    let category = '課程通知';
    let pinned = false;

    // 支援多標籤解析（如 [收費通知] [置頂]）
    const tagMatches = title.match(/\[(.*?)\]/g);
    if (tagMatches) {
      tagMatches.forEach(function(m) {
        const tag = m.replace(/[\[\]]/g, '').trim();
        if (tag === '置頂') {
          pinned = true;
        } else if (APP.categories.indexOf(tag) !== -1) {
          category = tag;
        }
        title = title.replace(m, '').trim();
      });
    }

    const todayStr = Utilities.formatDate(new Date(), APP.timezone, 'yyyy-MM-dd');
    const noticePayload = {
      title: title || '重要通知',
      noticeDate: todayStr,
      category: category,
      content: content,
      imageFileId: '',
      imageUrl: '',
      published: true,
      pinned: pinned,
      startDate: '',
      endDate: ''
    };

    const savedNotice = saveNoticeInternal_(noticePayload);
    const broadcastRes = broadcastLineNotice_(savedNotice, token);
    const usage = getLineMonthlyUsage_(token);

    let replyMsg = '✅ 公告已成功發布並同步！\n\n' +
      '📌 標題：' + savedNotice.title + '\n' +
      '📂 分類：' + savedNotice.category + (savedNotice.pinned ? '（置頂）' : '') + '\n' +
      '🌐 雲端公告：已登錄\n' +
      '📡 LINE 群發：' + (broadcastRes.success ? '已推送至所有好友' : '推送失敗：' + broadcastRes.error) + '\n\n' +
      '💡 本月已用訊息數：' + usage.totalUsage + '/200 則';

    if (usage.totalUsage >= 180) {
      replyMsg += '\n⚠️ 提醒：免費額度即將用罄！';
    }

    replyLineMessage_(replyToken, replyMsg, token);
  } catch (err) {
    console.error('processTextNoticeFromLine_ error:', err);
    replyLineMessage_(replyToken, '❌ 發布失敗：' + err.message, token);
  }
}

function parseNoticeText_(raw) {
  let title = '';
  let content = '';

  // 1. 若有換行：第一行為標題，其餘為內文
  if (raw.indexOf('\n') !== -1) {
    const lines = raw.split('\n');
    title = lines[0].trim();
    content = lines.slice(1).join('\n').trim();
  }
  // 2. 若以 【標題】 格式開頭
  else if (/^【(.*?)】(.*)/s.test(raw)) {
    const m = raw.match(/^【(.*?)】(.*)/s);
    title = m[1].trim();
    content = m[2].trim();
  }
  // 3. 若包含 // 或 ｜ 或 | 分隔符號
  else if (/\s+(\/\/|\||｜)\s+/.test(raw)) {
    const parts = raw.split(/\s+(\/\/|\||｜)\s+/);
    title = parts[0].trim();
    content = parts.slice(2).join('').trim();
  }
  // 4. 若包含全形或半形冒號 ： / :
  else if (raw.indexOf('：') !== -1 || raw.indexOf(':') !== -1) {
    const idx = raw.indexOf('：') !== -1 ? raw.indexOf('：') : raw.indexOf(':');
    title = raw.slice(0, idx).trim();
    content = raw.slice(idx + 1).trim();
  }
  // 5. 單行無分隔符號：依字數長度智慧防呆
  else {
    if (raw.length <= 25) {
      title = raw;
      content = raw;
    } else {
      // 尋找前 30 字內的第一個標點符號作為切分點
      const punctMatch = raw.slice(0, 30).match(/[，,。!！?？]/);
      if (punctMatch && punctMatch.index >= 4) {
        title = raw.slice(0, punctMatch.index).trim();
        content = raw.slice(punctMatch.index + 1).trim();
      } else {
        title = raw.slice(0, 20).trim();
        content = raw.slice(20).trim();
      }
    }
  }

  if (!content) content = title;
  return { title: title, content: content };
}

function processImageNoticeFromLine_(messageId, replyToken, token) {
  try {
    // 取得 LINE 圖片二進位內容
    const imgUrl = 'https://api-data.line.me/v2/bot/message/' + encodeURIComponent(messageId) + '/content';
    const response = UrlFetchApp.fetch(imgUrl, {
      headers: { 'Authorization': 'Bearer ' + token },
      muteHttpExceptions: true
    });

    if (response.getResponseCode() !== 200) {
      throw new Error('無法從 LINE 下載圖片（代碼 ' + response.getResponseCode() + '）');
    }

    const blob = response.getBlob();
    const todayStr = Utilities.formatDate(new Date(), APP.timezone, 'yyyy-MM-dd');
    blob.setName('LINE圖片公告_' + todayStr + '_' + Utilities.getUuid().slice(0, 6) + '.jpg');

    const folder = ensureImageFolder_();
    const file = folder.createFile(blob);
    try {
      file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    } catch (e) {
      console.warn('Drive sharing warning:', e);
    }

    const fileId = file.getId();
    const imageUrl = driveImageUrl_(fileId);

    const noticePayload = {
      title: '圖片通知 - ' + todayStr,
      noticeDate: todayStr,
      category: '圖片通知',
      content: '（此通知為 LINE 官方帳號同步上傳之圖片）',
      imageFileId: fileId,
      imageUrl: imageUrl,
      published: true,
      pinned: false,
      startDate: '',
      endDate: ''
    };

    const savedNotice = saveNoticeInternal_(noticePayload);
    const broadcastRes = broadcastLineNotice_(savedNotice, token);
    const usage = getLineMonthlyUsage_(token);

    let replyMsg = '✅ 圖片公告已發布並同步！\n\n' +
      '📌 標題：' + savedNotice.title + '\n' +
      '🌐 雲端公告：已登錄\n' +
      '📡 LINE 群發：' + (broadcastRes.success ? '已推送至所有好友' : '推送失敗：' + broadcastRes.error) + '\n\n' +
      '💡 本月已用訊息數：' + usage.totalUsage + '/200 則';

    replyLineMessage_(replyToken, replyMsg, token);
  } catch (err) {
    console.error('processImageNoticeFromLine_ error:', err);
    replyLineMessage_(replyToken, '❌ 圖片處理失敗：' + err.message, token);
  }
}

function broadcastLineNotice_(notice, token) {
  try {
    const webAppUrl = ScriptApp.getService().getUrl() || '';
    const messages = [];

    if (notice.imageFileId) {
      const lineImg = 'https://lh3.googleusercontent.com/d/' + encodeURIComponent(notice.imageFileId);
      messages.push({
        type: 'image',
        originalContentUrl: lineImg,
        previewImageUrl: lineImg
      });
    }

    let textContent = '📢【' + notice.title + '】\n\n' +
      (notice.content ? notice.content + '\n\n' : '');
    if (webAppUrl) {
      textContent += '🌐 家長通知中心完整版：\n' + webAppUrl;
    }

    messages.push({
      type: 'text',
      text: textContent.trim()
    });

    const res = UrlFetchApp.fetch('https://api.line.me/v2/bot/message/broadcast', {
      method: 'post',
      contentType: 'application/json',
      headers: { 'Authorization': 'Bearer ' + token },
      payload: JSON.stringify({ messages: messages }),
      muteHttpExceptions: true
    });

    const code = res.getResponseCode();
    if (code !== 200) {
      return { success: false, error: 'HTTP ' + code + ' ' + res.getContentText() };
    }
    return { success: true };
  } catch (err) {
    console.error('broadcastLineNotice_ error:', err);
    return { success: false, error: err.message };
  }
}

function getLineMonthlyUsage_(token) {
  try {
    const res = UrlFetchApp.fetch('https://api.line.me/v2/bot/message/quota/consumption', {
      headers: { 'Authorization': 'Bearer ' + token },
      muteHttpExceptions: true
    });
    if (res.getResponseCode() === 200) {
      const data = JSON.parse(res.getContentText());
      return { totalUsage: data.totalUsage || 0 };
    }
  } catch (e) {
    console.error('getLineMonthlyUsage_ error:', e);
  }
  return { totalUsage: 0 };
}

function replyLineMessage_(replyToken, text, token) {
  if (!replyToken || !text) return;
  try {
    UrlFetchApp.fetch('https://api.line.me/v2/bot/message/reply', {
      method: 'post',
      contentType: 'application/json',
      headers: { 'Authorization': 'Bearer ' + token },
      payload: JSON.stringify({
        replyToken: replyToken,
        messages: [{ type: 'text', text: text }]
      }),
      muteHttpExceptions: true
    });
  } catch (e) {
    console.error('replyLineMessage_ error:', e);
  }
}

function getLineHelpText_() {
  return '📋【LINE 官方帳號公告發布說明】\n\n' +
    '1. 發布純文字公告：\n' +
    '#公告 9月親師座談會通知\n' +
    '各位家長好，座談會將於下週五舉行...\n\n' +
    '2. 指定分類：\n' +
    '#公告 [收費通知] 課外活動繳費\n' +
    '請於本週日前完成繳費...\n\n' +
    '3. 置頂公告：\n' +
    '#公告 [置頂] 開學必讀注意事項\n' +
    '內容...\n\n' +
    '4. 發布圖片公告：\n' +
    '直接傳送圖片給官方帳號即可！\n\n' +
    '5. 查詢用量：\n' +
    '傳送「#額度」可查詢本月剩餘免費則數。';
}
