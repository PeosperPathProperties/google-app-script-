// @ts-nocheck

/**
 * 1) onFormSubmit(e)
 *    – Logs raw values in Debug
 *    – Upserts into Users sheet
 *    – Immediately sends Day-0 “welcome” email+SMS (if underContract = Yes)
 *    – Ensures time-based triggers exist
 */
function onFormSubmit(e) {
  const ss  = SpreadsheetApp.getActive();

  // — 1a) Debug sheet: log raw values so we can see exactly what came in
  let dbg = ss.getSheetByName('Debug');
  if (!dbg) {
    dbg = ss.insertSheet('Debug');
    dbg.appendRow(['Timestamp','Payload']);
  }
  dbg.appendRow([ new Date(), e.values.join(' | ') ]);

  // — 1b) Parse incoming form values by position:
  //     [ Timestamp, Name, Email, Phone Number, Are you under contract?, Track ]
  const [ , name, email, phone, underContract, track ] = e.values;
  const uc = (underContract === 'Yes');
  if (!email) {
    // if no email, bail out
    return;
  }

  // — 2) Upsert into Users sheet
  let sh = ss.getSheetByName('Users');
  if (!sh) {
    sh = ss.insertSheet('Users');
    sh.appendRow([
      'Name','Email','Phone','Track',
      'UnderContract?','SubscribedOn','LastSentOn','IsUnsubscribed'
    ]);
  }

  // Pull all existing rows
  const data = sh.getDataRange().getValues(); // includes header row
  const idx  = data.findIndex((row, i) => i > 0 && row[1] === email);

  // If we found an existing email, update columns A–E:
  if (idx > 0) {
    sh.getRange(idx+1, 1, 1, 5)
      .setValues([[ name, email, phone, track, uc ]]);
  } else {
    // Otherwise append a new row A–H
    sh.appendRow([ name, email, phone, track, uc, new Date(), '', false ]);
  }

  // — 3) Immediately send a Day-0 Welcome if they are under contract
  //     Only send if they signed up today (we wrote `SubscribedOn = today` for brand-new).
  if (uc) {
    try {
      // Compute the “welcome” row (DAY = 0) from your track sheet
      const trSheet = ss.getSheetByName('JON ORIGINAL TRACK 1');
      const hdr     = trSheet.getRange(3, 1, 1, trSheet.getLastColumn()).getValues()[0];
      const DAY_C   = hdr.indexOf('DAY') + 1;
      const TXT_C   = hdr.indexOf('COMPILED TEXT') + 1;
      const HTML_C  = hdr.indexOf('COMPILED EMAIL') + 1;
      const rows    = trSheet.getRange(4, 1, trSheet.getLastRow() - 3, hdr.length).getValues();
      const welcomeRow = rows.find(r => Number(r[DAY_C - 1]) === 0);

      if (welcomeRow) {
        // Only send “Welcome” if subOn === today (brand-new subscriber)
        // Determine the newly set “SubscribedOn” from the sheet row we just wrote:
        let subOn;
        if (idx > 0) {
          // Existing row → read SubscribedOn from that row (column F)
          subOn = data[idx][5];
        } else {
          // Newly appended → we set subOn to “now”
          subOn = new Date();
        }

        // Compare subOn to today’s date string:
        if (subOn instanceof Date && subOn.toDateString() === new Date().toDateString()) {
          sendMessage(
            email,
            phone,
            welcomeRow[TXT_C - 1],   // SMS text
            welcomeRow[HTML_C - 1],  // EMAIL HTML
            `Welcome aboard, ${name}!` // override subject
          );
          // Log the “LastSentOn” timestamp
          const rowIndex = (idx > 0) ? idx + 1 : sh.getLastRow();
          sh.getRange(rowIndex, 7).setValue(new Date());
        }
      }
    } catch (err) {
      // If anything goes wrong, log the stack to Debug
      dbg.appendRow([ new Date(), 'Day 0 error:', err.stack ]);
    }
  }

  // — 4) Ensure your time-based triggers are in place
  ensureTriggers();
}


/**
 * 1a) ensureTriggers()
 *     – installs a daily sendDailyAssignments @ 09:00
 *     – installs a pullEmailResponses every 5 minutes
 */
function ensureTriggers() {
  const existing = ScriptApp.getProjectTriggers().map(t => t.getHandlerFunction());
  if (!existing.includes('sendDailyAssignments')) {
    ScriptApp.newTrigger('sendDailyAssignments')
             .timeBased().everyDays(1).atHour(9).create();
  }
  if (!existing.includes('pullEmailResponses')) {
    ScriptApp.newTrigger('pullEmailResponses')
             .timeBased().everyMinutes(5).create();
  }
}


function sendDailyAssignments() {
  const ss     = SpreadsheetApp.getActive();
  const today  = new Date();
  const uSheet = ss.getSheetByName('Users');
  const users  = uSheet.getDataRange().getValues().slice(1); // skip header
  const trSheet = ss.getSheetByName('JON ORIGINAL TRACK 1');

  // 1) Pull track headers & data once:
  const hdr     = trSheet.getRange(3,1,1,trSheet.getLastColumn()).getValues()[0];
  const DAY_C   = hdr.indexOf('DAY') + 1;
  const TXT_C   = hdr.indexOf('COMPILED TEXT') + 1;
  const HTML_C  = hdr.indexOf('COMPILED EMAIL') + 1;
  const allRows = trSheet.getRange(4,1,trSheet.getLastRow()-3, hdr.length).getValues();

  users.forEach((row,i) => {
    let [ name, email, phone, track, uc, subOn, lastSent, unsub ] = row;

    // 1A) If not under contract or unsubscribed → skip
    if (!uc || unsub) return;

    // 2) If subOn is exactly today → send Day 0 and log it
    if (subOn instanceof Date && subOn.toDateString() === today.toDateString()) {
      const welcomeRow = allRows.find(r => Number(r[DAY_C-1]) === 0);
      if (welcomeRow) {
        sendMessage(
          email, phone,
          welcomeRow[TXT_C-1],
          welcomeRow[HTML_C-1],
          `Welcome aboard, ${name}!`
        );
        uSheet.getRange(i+2, 7).setValue(new Date()); // update LastSentOn
      }
      return;
    }

    // 3) Otherwise, compute “weekIndex” since subOn:
    //    - Find the first Monday on-or-after subOn → call that “firstMon”
    //    - If today < firstMon, skip (it’s not yet a Monday for Day1)
    //    - Else count how many Mondays have _passed_ from firstMon up to today.
    //
    //    weekIndex = (# of full weeks that have elapsed since firstMon) + 1
    //
    //    Because weekIndex=1 → Day1, weekIndex=2 → Day2, … weekIndex=21 → Day21

    // A) Find “firstMon”:
    const firstMon = getNextMonday(subOn);  // helper: returns a Date
    if (today < firstMon) {
      // It’s not Monday1 yet. Don’t send anything this week.
      return;
    }

    // B) Compute full weeks between firstMon and today:
    //    -(today - firstMon)/(ms per day) gives raw days since first Monday
    //    - Divide by 7, floor → that many full weeks
    //    - Then + 1 → the “Day N” index
    const msPerDay = 1000*60*60*24;
    const daysSinceFirstMon = Math.floor((today - firstMon)/msPerDay);
    const weekIndex = Math.floor(daysSinceFirstMon / 7) + 1;

    // C) If weekIndex > 21, we have exhausted the sequence → skip
    if (weekIndex > 21) {
      // You could optionally mark them “completed” or set unsub=true
      return;
    }

    // D) If lastSent is also some Monday in the same “weekIndex,” skip.
    //    (This prevents multiple sends on the same Monday if the trigger runs multiple times.)
    //
    //    Let lastSentMondayIndex = # of full weeks since firstMon at the time lastSent.
    //    If lastSentMondayIndex >= weekIndex, it means we already sent that week’s email.
    if (lastSent instanceof Date) {
      const daysSinceFirstMonAtLastSent = Math.floor((lastSent - firstMon)/msPerDay);
      const lastSentWeekIndex = Math.floor(daysSinceFirstMonAtLastSent/7) + 1;
      if (lastSentWeekIndex >= weekIndex) {
        return;  // we already sent this exact Week’s message
      }
    }

    // 4) Find the matching row in track sheet: DAY == weekIndex
    const matching = allRows.find(r => Number(r[DAY_C-1]) === weekIndex);
    if (!matching) {
      // (You might want to log: “no row for day = weekIndex.”)
      return;
    }

    // 5) Finally send email + SMS for Day weekIndex
    sendMessage(
      email, phone,
      matching[TXT_C-1],
      matching[HTML_C-1],
      `Your Day ${weekIndex} Assignments`
    );
    uSheet.getRange(i+2,7).setValue(new Date());  // update LastSentOn to now
  });
}

/**
 * Returns the first Monday on-or-after the given date.
 */
function getNextMonday(d) {
  // Clone ’d’ (to avoid mutating original)
  const out = new Date(d);
  // If ’d’ is already Monday (getDay()===1), return that same date
  // Else add whatever days it takes to reach next Monday
  const dayOfWeek = out.getDay();  // 0=Sun, 1=Mon, ..., 6=Sat
  const daysUntilMon = (dayOfWeek === 1) ? 0 : ((8 - dayOfWeek) % 7);
  out.setDate(out.getDate() + daysUntilMon);
  out.setHours(0,0,0,0);  // normalize to midnight
  return out;
}


/**
 * 3) pullEmailResponses()
 *    – stub for processing incoming emails (e.g. “UNSUBSCRIBE” replies)
 */
function pullEmailResponses() {
  // (you can implement this later, if you want to process inbound replies)
}


/**
 * Helper to send BOTH email & (optionally) SMS via Twilio.
 */
function sendMessage(toEmail, toPhone, smsText, htmlBody, subject) {
  // ==== SEND EMAIL ====
  MailApp.sendEmail({
    to:       toEmail,
    subject:  subject,
    htmlBody: htmlBody || smsText,
    body:     smsText
  });

  // ==== SEND SMS? ====
  const props = PropertiesService.getScriptProperties();
  if (props.getProperty('ENABLE_SMS') === 'true') {
    const SID  = props.getProperty('TWILIO_SID');
    const TOK  = props.getProperty('TWILIO_TOKEN');
    const FROM = props.getProperty('TWILIO_FROM');
    if (SID && TOK && FROM && toPhone) {
      const authHdr = 'Basic ' + Utilities.base64Encode(`${SID}:${TOK}`);
      UrlFetchApp.fetch(
        `https://api.twilio.com/2010-04-01/Accounts/${SID}/Messages.json`,
        {
          method: 'post',
          headers: { Authorization: authHdr },
          payload: {
            To:   toPhone.startsWith('+') ? toPhone : '+' + toPhone,
            From: FROM,
            Body: smsText
          }
        }
      );
    }
  }
}


/**
 * 4) previewDay()
 *    – Prompts you for a single day (0–21), then sends you an email preview of that day’s template.
 *    – (SMS preview can be uncommented if desired.)
 */
function previewDay() {
  const ui   = SpreadsheetApp.getUi();
  const resp = ui.prompt(
    'Preview Day',
    'Enter day number (0 for welcome, 1 for Monday, 2 for Tuesday, …):',
    ui.ButtonSet.OK_CANCEL
  );
  if (resp.getSelectedButton() !== ui.Button.OK) return;
  const dayNum = Number(resp.getResponseText());
  if (isNaN(dayNum)) {
    ui.alert('⚠️ Please enter a valid numeric day between 0 and 21.');
    return;
  }

  const ss    = SpreadsheetApp.getActive();
  const me    = Session.getEffectiveUser().getEmail();
  const tr    = ss.getSheetByName('JON ORIGINAL TRACK 1');
  const hdr   = tr.getRange(3, 1, 1, tr.getLastColumn()).getValues()[0];
  const DAY_C = hdr.indexOf('DAY')           + 1;
  const TXT_C = hdr.indexOf('COMPILED TEXT') + 1;
  const HTML_C= hdr.indexOf('COMPILED EMAIL')+ 1;
  const rows  = tr.getRange(4, 1, tr.getLastRow() - 3, hdr.length).getValues();
  const row   = rows.find(r => Number(r[DAY_C - 1]) === dayNum);
  if (!row) {
    ui.alert(`⚠️ No template found for DAY = ${dayNum}.`);
    return;
  }

  const smsText  = row[TXT_C - 1]  || '(no COMPILED TEXT)';
  const htmlBody = row[HTML_C - 1] || '<i>(no COMPILED EMAIL)</i>';

  MailApp.sendEmail({
    to:       me,
    subject:  `📬 Preview: Day ${dayNum} SMS + Email`,
    htmlBody: `<pre style="white-space:pre-wrap;">${smsText}</pre><hr>${htmlBody}`,
    body:     smsText
  });
  ui.alert(`✅ Sent Day ${dayNum} preview to ${me}.`);
}


// @ts-nocheck

/**
 * 1) onFormSubmit(e)
 *    – Logs raw values in Debug
 *    – Upserts into Users sheet
 *    – Immediately sends Day-0 “welcome” email+SMS (if underContract = Yes)
 *    – Ensures time-based triggers exist
 */
function onFormSubmit(e) {
  const ss  = SpreadsheetApp.getActive();

  // — 1a) Debug sheet: log raw values so we can see exactly what came in
  let dbg = ss.getSheetByName('Debug');
  if (!dbg) {
    dbg = ss.insertSheet('Debug');
    dbg.appendRow(['Timestamp','Payload']);
  }
  dbg.appendRow([ new Date(), e.values.join(' | ') ]);

  // — 1b) Parse incoming form values by position:
  //     [ Timestamp, Name, Email, Phone Number, Are you under contract?, Track ]
  const [ , name, email, phone, underContract, track ] = e.values;
  const uc = (underContract === 'Yes');
  if (!email) {
    // if no email, bail out
    return;
  }

  // — 2) Upsert into Users sheet
  let sh = ss.getSheetByName('Users');
  if (!sh) {
    sh = ss.insertSheet('Users');
    sh.appendRow([
      'Name','Email','Phone','Track',
      'UnderContract?','SubscribedOn','LastSentOn','IsUnsubscribed'
    ]);
  }

  // Pull all existing rows
  const data = sh.getDataRange().getValues(); // includes header row
  const idx  = data.findIndex((row, i) => i > 0 && row[1] === email);

  // If we found an existing email, update columns A–E:
  if (idx > 0) {
    sh.getRange(idx+1, 1, 1, 5)
      .setValues([[ name, email, phone, track, uc ]]);
  } else {
    // Otherwise append a new row A–H
    sh.appendRow([ name, email, phone, track, uc, new Date(), '', false ]);
  }

  // — 3) Immediately send a Day-0 Welcome if they are under contract
  //     Only send if they signed up today (we wrote `SubscribedOn = today` for brand-new).
  if (uc) {
    try {
      // Compute the “welcome” row (DAY = 0) from your track sheet
      const trSheet = ss.getSheetByName('JON ORIGINAL TRACK 1');
      const hdr     = trSheet.getRange(3, 1, 1, trSheet.getLastColumn()).getValues()[0];
      const DAY_C   = hdr.indexOf('DAY') + 1;
      const TXT_C   = hdr.indexOf('COMPILED TEXT') + 1;
      const HTML_C  = hdr.indexOf('COMPILED EMAIL') + 1;
      const rows    = trSheet.getRange(4, 1, trSheet.getLastRow() - 3, hdr.length).getValues();
      const welcomeRow = rows.find(r => Number(r[DAY_C - 1]) === 0);

      if (welcomeRow) {
        // Only send “Welcome” if subOn === today (brand-new subscriber)
        // Determine the newly set “SubscribedOn” from the sheet row we just wrote:
        let subOn;
        if (idx > 0) {
          // Existing row → read SubscribedOn from that row (column F)
          subOn = data[idx][5];
        } else {
          // Newly appended → we set subOn to “now”
          subOn = new Date();
        }

        // Compare subOn to today’s date string:
        if (subOn instanceof Date && subOn.toDateString() === new Date().toDateString()) {
          sendMessage(
            email,
            phone,
            welcomeRow[TXT_C - 1],   // SMS text
            welcomeRow[HTML_C - 1],  // EMAIL HTML
            `Welcome aboard, ${name}!` // override subject
          );
          // Log the “LastSentOn” timestamp
          const rowIndex = (idx > 0) ? idx + 1 : sh.getLastRow();
          sh.getRange(rowIndex, 7).setValue(new Date());
        }
      }
    } catch (err) {
      // If anything goes wrong, log the stack to Debug
      dbg.appendRow([ new Date(), 'Day 0 error:', err.stack ]);
    }
  }

  // — 4) Ensure your time-based triggers are in place
  ensureTriggers();
}


/**
 * 1a) ensureTriggers()
 *     – installs a daily sendDailyAssignments @ 09:00
 *     – installs a pullEmailResponses every 5 minutes
 */
function ensureTriggers() {
  const existing = ScriptApp.getProjectTriggers().map(t => t.getHandlerFunction());
  if (!existing.includes('sendDailyAssignments')) {
    ScriptApp.newTrigger('sendDailyAssignments')
             .timeBased().everyDays(1).atHour(9).create();
  }
  if (!existing.includes('pullEmailResponses')) {
    ScriptApp.newTrigger('pullEmailResponses')
             .timeBased().everyMinutes(5).create();
  }
}


/**
 * Prompt for a day-number (0–21), then SMS that day’s COMPILED TEXT to your own mobile.
 * Relies on sendSmsOnly(toPhone, smsText) rather than sendMessage().
 */
function previewSmsOnly() {
  const ui = SpreadsheetApp.getUi();
  const resp = ui.prompt(
    'Preview SMS Only',
    'Enter day number (0 for Welcome, 1 for Monday, …):',
    ui.ButtonSet.OK_CANCEL
  );
  if (resp.getSelectedButton() !== ui.Button.OK) return;

  const dayNum = Number(resp.getResponseText());
  if (isNaN(dayNum) || dayNum < 0) {
    ui.alert('❌ Please enter a valid non-negative integer (e.g. 0, 1, 2, …).');
    return;
  }

  // 1) Grab your “JON ORIGINAL TRACK 1” sheet & find the header:
  const ss = SpreadsheetApp.getActive();
  const tr = ss.getSheetByName('JON ORIGINAL TRACK 1');
  if (!tr) {
    ui.alert('❌ Could not find sheet named "JON ORIGINAL TRACK 1".');
    return;
  }
  const hdr = tr.getRange(3, 1, 1, tr.getLastColumn()).getValues()[0];
  const DAY_C = hdr.indexOf('DAY') + 1;
  const TXT_C = hdr.indexOf('COMPILED TEXT') + 1;
  if (DAY_C === 0 || TXT_C === 0) {
    ui.alert('❌ Header row must contain "DAY" and "COMPILED TEXT" exactly (caps matter).');
    return;
  }

  // 2) Pull every data row (row 4 → last)
  const rowCount = tr.getLastRow() - 3;
  if (rowCount <= 0) {
    ui.alert('❌ No data rows found in "JON ORIGINAL TRACK 1".');
    return;
  }
  const rows = tr.getRange(4, 1, rowCount, tr.getLastColumn()).getValues();

  // 3) Find the matching row for dayNum
  const row = rows.find(r => Number(r[DAY_C - 1]) === dayNum);
  if (!row) {
    ui.alert(`❌ No template found for DAY = ${dayNum}.`);
    return;
  }

  // 4) Extract that day’s COMPILED TEXT
  const smsText = String(row[TXT_C - 1] || '').trim();
  if (!smsText) {
    ui.alert(`⚠️ Found DAY ${dayNum}, but the COMPILED TEXT cell is blank.`);
    return;
  }

  // 5) Prompt for the “To” phone in E.164
  const pResp = ui.prompt(
    'Send SMS To (E.164)',
    'Enter your phone number in E.164 format (e.g. +12025551234):',
    ui.ButtonSet.OK_CANCEL
  );
  if (pResp.getSelectedButton() !== ui.Button.OK) return;

  const rawPhone = pResp.getResponseText().trim();
  if (!rawPhone.startsWith('+') || rawPhone.length < 8) {
    ui.alert('❌ That does not look like a valid E.164 phone (it must start with “+”).');
    return;
  }

  // 6) Call sendSmsOnly(toPhone, smsText)
  try {
    sendSmsOnly(rawPhone, smsText);
    ui.alert(`✅ SMS for Day ${dayNum} sent to ${rawPhone}. Check your phone now.`);
  } catch (err) {
    ui.alert(`❌ Error sending SMS: ${err.message}`);
  }
}
