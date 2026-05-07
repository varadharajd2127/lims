# AZBT LIMS — Complete Change Specification
**File:** Changes required in `booking.html`, `admin.html`, and `code.gs`  
**Format:** File → Location → What to change / add

---

## 1. SLOT DISPLAY BUG — Already-booked slots still show as bookable

### `booking.html` → `loadSlots()` function (~line 666)

**Problem:** `mySlots` is filtered from `myBookings` but `busySlots` also includes the user's own slots, so the `mine` check can still be overridden. More critically, a slot that is `Approved` for any user (including the current user) is still shown as clickable if the duration overlap logic isn't run per-slot. The current code only checks `b.time === slot` — it misses multi-hour overlaps (e.g. a 3-hour booking at 09:00 should block 09, 10, 11).

**Fix — replace the entire `loadSlots()` function:**

```js
function loadSlots(){
  const instrId = document.getElementById('bInstr').value;
  const date    = document.getElementById('bDate').value;
  const sec     = document.getElementById('slotSection');
  selectedSlot  = null;
  if(!instrId || !date){ sec.style.display='none'; return; }
  const instr = instruments.find(i => i.id === instrId);
  if(!instr){ sec.style.display='none'; return; }
  sec.style.display = 'block';

  const slots = [];
  for(let h = instr.startHour; h < instr.endHour; h++) slots.push(pad(h)+':00');

  // Build per-slot occupancy using duration overlap
  const activeBooks = allBookings.filter(b =>
    b.instrId === instrId && b.date === date && b.status !== 'Rejected'
  );

  document.getElementById('slotGrid').innerHTML = slots.map(slot => {
    const slotStart = timeToM(slot);
    const slotEnd   = slotStart + 60; // each slot = 1 hour

    // Find any booking that overlaps this slot
    const overlap = activeBooks.find(b => {
      const bs = timeToM(b.time), be = bs + (b.dur * 60);
      return bs < slotEnd && slotStart < be;
    });

    if(overlap){
      if(overlap.userId === user.uid){
        // Current user's own booking — show "Yours" with cancel option
        return `<div class="slot-btn mine" title="Your booking: ${overlap.id}&#10;Status: ${overlap.status}&#10;Click to cancel"
          onclick="cancelSlotBooking('${overlap.id}')">
          ${slot}<br><span style="font-size:9px;"><i class="fa-solid fa-user-check"></i> Yours</span>
        </div>`;
      } else {
        // Another user's booking
        return `<div class="slot-btn busy" title="Booked">
          ${slot}<br><span style="font-size:9px;"><i class="fa-solid fa-lock"></i> Booked</span>
        </div>`;
      }
    }
    return `<div class="slot-btn" onclick="selectSlot(this,'${slot}')">${slot}</div>`;
  }).join('');
}
```

---

## 2. USER CANCEL BOOKING — From slot grid and history table

### `booking.html` → Add new function after `selectSlot()`

```js
function cancelSlotBooking(bkId){
  if(!confirm('Cancel your booking ' + bkId + '?')) return;
  fetch(SCRIPT_URL, {
    method:'POST',
    body: JSON.stringify({ action:'cancelBooking', bookingId: bkId })
  }).then(r => r.json()).then(res => {
    if(res.status === 'ok'){
      toast('Booking cancelled.', 'ok');
      fetchData();
    } else {
      toast(res.msg || 'Cancel failed.', 'err');
    }
  });
}
```

### `booking.html` → `bookingRow()` function (~line 576)

Add a **Cancel** button for Pending or Approved future bookings. Replace the function:

```js
function bookingRow(b){
  const today = new Date().toISOString().split('T')[0];
  const canCancel = (b.status === 'Pending' || b.status === 'Approved') && b.date >= today;
  return `<tr>
    <td><code style="font-size:11px;font-weight:700;color:var(--blue);">${b.id}</code></td>
    <td style="font-size:12.5px;">${b.instrName}</td>
    <td>${b.date}</td>
    <td>${formatTime(b.time)}</td>
    <td>${b.dur}h</td>
    <td style="max-width:130px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font-size:12px;">${b.purpose}</td>
    <td style="font-size:12px;">${b.guide}</td>
    <td><span class="badge b-${b.status.toLowerCase()}">${b.status}</span></td>
    <td>
      <button class="btn btn-blue btn-xs" onclick="printSingleBooking('${b.id}')" title="Print">
        <i class="fa-solid fa-print"></i>
      </button>
      ${canCancel ? `<button class="btn btn-red btn-xs" onclick="cancelSlotBooking('${b.id}')" title="Cancel">
        <i class="fa-solid fa-xmark"></i>
      </button>` : ''}
    </td>
  </tr>`;
}
```

> Also update the `<thead>` row in `historyPanel` to add a **Actions** column:  
> Add `<th>Actions</th>` after the last `<th>Status</th>`.  
> Do the same for the `recentBody` table header in the profile panel.

---

## 3. TIME FORMAT — 12-hour AM/PM display everywhere

### `booking.html` → Add helper function in the `/* ── UTILS ──` section

```js
function formatTime(t){
  if(!t) return '—';
  const [hStr, mStr] = t.split(':');
  let h = parseInt(hStr), m = parseInt(mStr)||0;
  const ampm = h >= 12 ? 'PM' : 'AM';
  h = h % 12 || 12;
  return `${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')} ${ampm}`;
}
```

### `booking.html` — Apply `formatTime()` everywhere a time is displayed:
- In `bookingRow()` → `<td>${formatTime(b.time)}</td>` (done above)
- In `buildCalendar()` → inside `calClick` tooltip if added
- In slot buttons → replace `pad(h)+':00'` display with `formatTime(pad(h)+':00')` for **labels only** (keep the value as 24h for logic)

### `admin.html` → Add the same `formatTime()` function to the `<script>` section

Apply it in:
- `buildBookingTable()` row rendering → `b.time` display
- `printBooking()` → time field
- `showDayModal()` → `b.time` display
- `printAllBookings()` row rendering

---

## 4. CALENDAR FIX — booking.html calendar not working properly

### Root cause:
`buildCalendar()` is called before `allBookings` is populated (race condition: `fetchData()` makes two separate `fetch()` calls and calendar can render on an empty array). Also `populateCalFilter()` is called before instruments are loaded.

### Fix in `booking.html` → `fetchData()` function (~line 533):

Chain the two fetches so calendar builds only after both complete:

```js
function fetchData(){
  Promise.all([
    fetch(SCRIPT_URL+'?action=getInstruments').then(r=>r.json()),
    fetch(SCRIPT_URL+'?action=getBookings&uid='+encodeURIComponent(user.uid)).then(r=>r.json())
  ]).then(([iRes, bRes]) => {
    instruments  = (iRes.status==='ok'  ? iRes.instruments  : []);
    myBookings   = (bRes.status==='ok'  ? bRes.myBookings   : []);
    allBookings  = (bRes.status==='ok'  ? bRes.allBookings  : []);
    populateBookingSelect();
    populateCalFilter();
    initInstrGrid();
    setTodayDate();
    initProfile();
    checkNotifications(); // see section 8
  }).catch(() => {
    instruments=[]; myBookings=[]; allBookings=[];
    populateBookingSelect(); populateCalFilter(); initInstrGrid(); setTodayDate(); initProfile();
  });
}
```

### Fix in `booking.html` → `buildCalendar()` (~line 744):

The calendar currently only shows events but clicking a day just jumps to new-booking. Add a **day detail view** — replace `calClick`:

```js
function calClick(date){
  const dayBooks = allBookings.filter(b => b.date === date && b.status !== 'Rejected');
  if(dayBooks.length){
    // Show mini popup: list of bookings for that day
    const lines = dayBooks.map(b =>
      `• ${b.instrName} @ ${formatTime(b.time)} (${b.dur}h) — ${b.userId===user.uid?'<strong>Yours</strong>':b.status}`
    ).join('<br>');
    // Use existing toast or a simple alert; ideally add a small modal (see section 9)
    showCalDayPopup(date, lines);
  } else {
    // Day is free — pre-fill date and go to booking
    document.getElementById('bDate').value = date;
    switchPanel('newbooking');
    loadSlots();
  }
}

function showCalDayPopup(date, html){
  // Reuse any existing modal or add this lightweight one (see HTML addition below)
  document.getElementById('calDayDate').textContent = date;
  document.getElementById('calDayBody').innerHTML   = html;
  openModal('calDayModal');
}
```

### Add modal HTML for calendar day detail inside `booking.html` body (before closing `</body>`):

```html
<!-- Calendar Day Modal -->
<div class="modal-bg" id="calDayModal" onclick="closeModal('calDayModal')">
  <div class="modal-box" onclick="event.stopPropagation()">
    <div class="modal-header">
      <span id="calDayDate"></span>
      <button class="modal-close" onclick="closeModal('calDayModal')">×</button>
    </div>
    <div style="padding:14px;font-size:13px;line-height:1.8;" id="calDayBody"></div>
    <div style="padding:0 14px 14px;text-align:right;">
      <button class="btn btn-gold btn-sm" onclick="closeModal('calDayModal')">Close</button>
    </div>
  </div>
</div>
```

> Also add `openModal` / `closeModal` helpers if not already present in `booking.html`:
```js
function openModal(id){ document.getElementById(id).style.display='flex'; }
function closeModal(id){ document.getElementById(id).style.display='none'; }
```

### Admin calendar fix — `admin.html` → `buildCalendar()` (~line 1216):

Same race condition exists. Admin's `fetchAll()` (wherever both data sets are loaded) must use `Promise.all` similarly. Wrap `buildCalendar()` call inside `switchPanel()` with a guard:

```js
if(pid==='calendar' && allBookings.length > 0) buildCalendar();
else if(pid==='calendar') { fetchAll().then(()=>buildCalendar()); }
```

---

## 5. PRINT — Per-booking print for users + dual logo

### `booking.html` → Add `printSingleBooking()` function

```js
function printSingleBooking(bkId){
  const b = myBookings.find(x => x.id === bkId);
  if(!b){ toast('Booking not found','err'); return; }
  const w = window.open('','_blank');
  w.document.write(`<!DOCTYPE html><html><head><title>Booking ${bkId}</title>
  <style>
    body{font-family:Arial,sans-serif;margin:30px;color:#1E3057;}
    .header{display:flex;align-items:center;justify-content:space-between;border-bottom:2px solid #C9973A;padding-bottom:12px;margin-bottom:20px;}
    .logos{display:flex;align-items:center;gap:12px;}
    .logos img{height:56px;object-fit:contain;}
    .title{text-align:center;}
    h2{margin:0;font-size:18px;color:#1E3057;}
    .sub{font-size:11px;color:#555;margin-top:3px;}
    table{width:100%;border-collapse:collapse;font-size:13px;margin-top:10px;}
    td{padding:9px 10px;border-bottom:1px solid #e5e7eb;vertical-align:top;}
    td:first-child{font-weight:700;color:#1E3057;width:40%;}
    .status{display:inline-block;padding:3px 10px;border-radius:6px;font-size:12px;font-weight:700;}
    .s-approved{background:#dcfce7;color:#166534;}
    .s-pending{background:#fef9c3;color:#854d0e;}
    .s-rejected{background:#fee2e2;color:#991b1b;}
    .watermark{position:fixed;top:50%;left:50%;transform:translate(-50%,-50%) rotate(-30deg);
      font-size:80px;color:rgba(30,48,87,.05);font-weight:900;pointer-events:none;z-index:0;}
    .footer{text-align:center;margin-top:28px;font-size:11px;color:#888;border-top:1px solid #ddd;padding-top:12px;}
    @media print{body{margin:10px;}}
  </style></head><body>
  <div class="watermark">AZBT LIMS</div>
  <div class="header">
    <div class="logos">
      <img src="https://upload.wikimedia.org/wikipedia/en/thumb/1/10/Loyola_College_Chennai_logo.png/100px-Loyola_College_Chennai_logo.png" alt="Loyola"/>
      <!-- Replace both src values with your actual hosted logo URLs -->
      <img src="YOUR_DEPT_LOGO_URL" alt="AZBT"/>
    </div>
    <div class="title">
      <h2>AZBT LIMS — Booking Receipt</h2>
      <div class="sub">PG &amp; Research Dept. of Advanced Zoology &amp; Biotechnology<br>Loyola College, Chennai — 600 034</div>
    </div>
    <div style="font-size:11px;color:#888;text-align:right;">
      Printed: ${new Date().toLocaleDateString('en-IN',{day:'2-digit',month:'long',year:'numeric'})}
    </div>
  </div>
  <table>
    <tr><td>Booking ID</td><td><strong>${b.id}</strong></td></tr>
    <tr><td>User</td><td>${user.name} (${user.uid})</td></tr>
    <tr><td>Email</td><td>${user.email}</td></tr>
    <tr><td>Role / Class</td><td>${user.role}${user.cls?' · '+user.cls:''}</td></tr>
    <tr><td>Instrument</td><td>${b.instrName}</td></tr>
    <tr><td>Date</td><td>${b.date}</td></tr>
    <tr><td>Time</td><td>${formatTime(b.time)}</td></tr>
    <tr><td>Duration</td><td>${b.dur} hour(s)</td></tr>
    <tr><td>Purpose</td><td>${b.purpose}</td></tr>
    <tr><td>Guide / Supervisor</td><td>${b.guide}</td></tr>
    <tr><td>Notes</td><td>${b.notes||'—'}</td></tr>
    <tr><td>Status</td><td><span class="status s-${b.status.toLowerCase()}">${b.status}</span></td></tr>
    <tr><td>Submitted At</td><td>${b.createdAt||'—'}</td></tr>
  </table>
  <div class="footer">AZBT LIMS, Loyola College · hodzoology@loyolacollege.edu</div>
  </body></html>`);
  w.document.close(); w.print();
}
```

### `booking.html` → `printMyBookings()` (~line 801) — Add dual logos:

Replace the `<h2>` and opening block with:

```js
// Inside the w.document.write template, replace the <h2> header section with:
`<div style="display:flex;align-items:center;justify-content:space-between;border-bottom:2px solid #C9973A;padding-bottom:12px;margin-bottom:16px;">
  <div style="display:flex;align-items:center;gap:12px;">
    <img src="YOUR_COLLEGE_LOGO_URL" style="height:52px;object-fit:contain;" alt="Loyola"/>
    <img src="YOUR_DEPT_LOGO_URL"    style="height:52px;object-fit:contain;" alt="AZBT"/>
  </div>
  <div style="text-align:center;">
    <h2 style="margin:0;font-size:17px;color:#1E3057;">AZBT LIMS — My Booking History</h2>
    <div style="font-size:11px;color:#555;margin-top:3px;">PG &amp; Research Dept. of Advanced Zoology &amp; Biotechnology, Loyola College</div>
  </div>
  <div style="font-size:11px;color:#888;">Printed: ${new Date().toLocaleDateString('en-IN',{day:'2-digit',month:'long',year:'numeric'})}</div>
</div>`
```

### `admin.html` → `printBooking()` and `printHeader()` — Add dual logos:

In `printHeader()` (~line 1312), replace the static heading block with the same dual-logo structure shown above.

---

## 6. INSTRUMENT ISSUE REPORT — After each booking / standalone

### `booking.html` — Add a **Report Issue** button in `bookingRow()`:

Add alongside the print/cancel buttons (only for Approved past bookings):
```js
const isPast = b.date < new Date().toISOString().split('T')[0];
const canReport = b.status === 'Approved' && isPast;
// In the td:
${canReport ? `<button class="btn btn-red btn-xs" onclick="openReportModal('${b.id}','${b.instrId}','${b.instrName}')" title="Report Issue">
  <i class="fa-solid fa-triangle-exclamation"></i>
</button>` : ''}
```

### `booking.html` — Add Report Issue modal HTML (before `</body>`):

```html
<!-- Report Issue Modal -->
<div class="modal-bg" id="reportModal" onclick="closeModal('reportModal')">
  <div class="modal-box" onclick="event.stopPropagation()">
    <div class="modal-header">
      <span><i class="fa-solid fa-triangle-exclamation" style="color:var(--maroon);"></i> Report Instrument Issue</span>
      <button class="modal-close" onclick="closeModal('reportModal')">×</button>
    </div>
    <div style="padding:16px;">
      <input type="hidden" id="rptBkId"/><input type="hidden" id="rptInstrId"/>
      <div style="font-size:13px;font-weight:600;color:var(--blue);margin-bottom:12px;" id="rptInstrName"></div>
      <div class="form-group">
        <label class="form-label"><i class="fa-solid fa-tag"></i> Issue Type</label>
        <select id="rptType" class="form-select">
          <option value="Not Working">Not Working</option>
          <option value="Partial Malfunction">Partial Malfunction</option>
          <option value="Calibration Needed">Calibration Needed</option>
          <option value="Physical Damage">Physical Damage</option>
          <option value="Other">Other</option>
        </select>
      </div>
      <div class="form-group">
        <label class="form-label"><i class="fa-solid fa-comment"></i> Description <span class="req">*</span></label>
        <textarea id="rptDesc" class="form-input" rows="4" placeholder="Describe the issue in detail…" style="resize:vertical;"></textarea>
      </div>
      <div id="rptMsg"></div>
    </div>
    <div style="padding:0 16px 16px;display:flex;justify-content:flex-end;gap:8px;">
      <button class="btn btn-gray btn-sm" onclick="closeModal('reportModal')">Cancel</button>
      <button class="btn btn-red btn-sm" id="rptSubmitBtn" onclick="submitReport()">
        <i class="fa-solid fa-paper-plane"></i> Submit Report
      </button>
    </div>
  </div>
</div>
```

### `booking.html` — Add JS functions for report:

```js
function openReportModal(bkId, instrId, instrName){
  document.getElementById('rptBkId').value    = bkId;
  document.getElementById('rptInstrId').value = instrId;
  document.getElementById('rptInstrName').textContent = 'Instrument: ' + instrName;
  document.getElementById('rptDesc').value    = '';
  setMsg('rptMsg','');
  openModal('reportModal');
}

function submitReport(){
  const bkId    = document.getElementById('rptBkId').value;
  const instrId = document.getElementById('rptInstrId').value;
  const type    = document.getElementById('rptType').value;
  const desc    = document.getElementById('rptDesc').value.trim();
  if(!desc){ setMsg('rptMsg','Please describe the issue.'); return; }
  const btn = document.getElementById('rptSubmitBtn');
  btn.disabled = true; btn.innerHTML = '<span class="spinner"></span> Submitting…';
  fetch(SCRIPT_URL,{
    method:'POST',
    body: JSON.stringify({ action:'reportIssue', bookingId:bkId, instrId, userId:user.uid,
      userName:user.name, issueType:type, description:desc })
  }).then(r=>r.json()).then(res=>{
    btn.disabled=false; btn.innerHTML='<i class="fa-solid fa-paper-plane"></i> Submit Report';
    if(res.status==='ok'){
      closeModal('reportModal');
      toast('Issue reported. Admin has been notified.','ok');
    } else { setMsg('rptMsg', res.msg||'Failed.'); }
  });
}
```

### `code.gs` — Add `reportIssue` action in `doPost()`:

```js
else if (action === 'reportIssue') result = handleReportIssue(data);
```

### `code.gs` — Add `handleReportIssue()` function:

```js
function handleReportIssue(data) {
  if (!data.instrId || !data.description)
    return {status:'error', msg:'Missing fields.'};

  // Log to a new "IssueReports" sheet (create manually or via setupSheets)
  var sh = getSheet('IssueReports');
  var rptId = 'RPT' + String(sh.getLastRow()).padStart(4,'0');
  sh.appendRow([
    rptId, data.bookingId||'', data.instrId, data.userId||'',
    data.userName||'', data.issueType||'Other', data.description, 'Open', nowStr()
  ]);

  // Update instrument status to "Maintenance" if issue type is severe
  if(['Not Working','Physical Damage'].indexOf(data.issueType) > -1){
    var iSh = getSheet(SH_INSTRUMENTS);
    var row = findRowByField(iSh, 'InstrID', data.instrId);
    if(row > 0){
      var hm = getHeaderMap(iSh);
      iSh.getRange(row, hm['Status']).setValue('Maintenance');
    }
  }

  // Notify admin
  try {
    MailApp.sendEmail({
      to: ADMIN_EMAIL,
      subject: '[AZBT LIMS] Instrument Issue Report — ' + data.instrId,
      body: 'A user has reported an issue with an instrument.\n\n' +
            'Report ID: ' + rptId + '\n' +
            'Instrument: ' + data.instrId + '\n' +
            'Booking ID: ' + (data.bookingId||'—') + '\n' +
            'Reported by: ' + (data.userName||data.userId) + '\n' +
            'Issue Type: ' + data.issueType + '\n' +
            'Description: ' + data.description + '\n\n' +
            'Reported at: ' + nowStr()
    });
  } catch(e) {}

  return {status:'ok', rptId: rptId};
}
```

### `code.gs` → `setupSheets()` — Add IssueReports sheet creation:

```js
var rpt = ss.getSheetByName('IssueReports') || ss.insertSheet('IssueReports');
rpt.clearContents();
var rHeaders = ['ReportID','BookingID','InstrID','UserID','UserName','IssueType','Description','Status','ReportedAt'];
rpt.getRange(1,1,1,rHeaders.length).setValues([rHeaders]);
formatHeader(rpt, rHeaders.length);
```

### `admin.html` — Add Issue Reports tab/section:

Add a nav tab button `<button class="nav-tab" onclick="switchPanel('issues')">...Issue Reports</button>` and a panel that fetches and displays all reports with a "Mark Resolved" action. Add `issues` to the `switchPanel` array. Fetch via a new `getIssueReports` GET action in `code.gs` (mirror `handleGetAllBookings`).

---

## 7. NOTIFICATION BELL — Registration and booking status alerts

### `booking.html` — Add bell icon in the top navigation bar:

Find the nav bar HTML (the `<nav>` or `.topbar` element) and add before the avatar:

```html
<div style="position:relative;margin-right:8px;">
  <button id="bellBtn" onclick="toggleNotifications()" style="background:none;border:none;cursor:pointer;color:#fff;font-size:18px;position:relative;">
    <i class="fa-solid fa-bell"></i>
    <span id="bellBadge" style="display:none;position:absolute;top:-4px;right:-4px;background:#e11d48;color:#fff;border-radius:50%;font-size:10px;width:16px;height:16px;display:none;align-items:center;justify-content:center;" id="bellBadge">0</span>
  </button>
  <div id="notifDropdown" style="display:none;position:absolute;right:0;top:38px;width:290px;background:#fff;border-radius:12px;box-shadow:0 8px 32px rgba(30,48,87,.18);z-index:999;max-height:320px;overflow-y:auto;">
    <div style="padding:10px 14px;font-weight:700;font-size:12px;color:var(--blue);border-bottom:1px solid #f0f0f0;">Notifications</div>
    <div id="notifList" style="padding:8px 0;"></div>
  </div>
</div>
```

### `booking.html` — Add `checkNotifications()` JS function:

```js
function checkNotifications(){
  const seen = JSON.parse(localStorage.getItem('limsSeenNotifs')||'[]');
  const notifs = [];

  // Check if any previously-Pending bookings are now Approved/Rejected
  myBookings.forEach(b => {
    const key = b.id + '_' + b.status;
    if((b.status === 'Approved' || b.status === 'Rejected') && !seen.includes(key)){
      notifs.push({ key, msg: `Booking <strong>${b.id}</strong> was <strong>${b.status}</strong>.`,
        type: b.status === 'Approved' ? 'ok' : 'err' });
    }
  });

  const badge = document.getElementById('bellBadge');
  if(notifs.length){
    badge.style.display = 'flex';
    badge.textContent   = notifs.length;
    document.getElementById('notifList').innerHTML = notifs.map(n =>
      `<div style="padding:9px 14px;font-size:12px;border-bottom:1px solid #f5f5f5;cursor:pointer;
        color:${n.type==='ok'?'#166534':'#991b1b'}"
        onclick="markNotifSeen('${n.key}')">
        <i class="fa-solid fa-${n.type==='ok'?'circle-check':'circle-xmark'}"></i> ${n.msg}
      </div>`).join('');
  } else {
    badge.style.display = 'none';
    document.getElementById('notifList').innerHTML =
      '<div style="padding:18px;text-align:center;color:#999;font-size:12px;">No new notifications</div>';
  }
}

function markNotifSeen(key){
  const seen = JSON.parse(localStorage.getItem('limsSeenNotifs')||'[]');
  if(!seen.includes(key)) seen.push(key);
  localStorage.setItem('limsSeenNotifs', JSON.stringify(seen));
  checkNotifications();
}

function toggleNotifications(){
  const d = document.getElementById('notifDropdown');
  d.style.display = d.style.display === 'none' ? 'block' : 'none';
}
```

> Call `checkNotifications()` at the end of `initProfile()` and after `fetchData()` resolves.

---

## 8. WAITLIST — User joins waitlist for a booked slot

### `booking.html` → `loadSlots()` — For busy (other-user) slots, add a "Waitlist" button:

Replace the `busy` slot rendering (inside the fixed `loadSlots()` above):

```js
if(overlap && overlap.userId !== user.uid){
  // Check if current user is already on waitlist for this slot
  const onWaitlist = (waitlistData[instrId+'_'+date+'_'+slot] === user.uid);
  return `<div class="slot-btn busy" style="cursor:pointer;" title="Booked — join waitlist"
    onclick="${onWaitlist ? '' : `joinWaitlist('${instrId}','${date}','${slot}','${overlap.id}')`}">
    ${slot}<br>
    <span style="font-size:9px;">
      ${onWaitlist
        ? '<i class="fa-solid fa-clock-rotate-left"></i> Waitlisted'
        : '<i class="fa-solid fa-list"></i> Join Waitlist'}
    </span>
  </div>`;
}
```

### `booking.html` — Add `waitlistData` variable and functions:

```js
let waitlistData = {}; // key: instrId_date_slot → userId (loaded from localStorage)

window.addEventListener('load', () => {
  // ... existing code
  waitlistData = JSON.parse(localStorage.getItem('limsWaitlist_'+user?.uid)||'{}');
});

function joinWaitlist(instrId, date, slot, occupiedBkId){
  if(!confirm('Join the waitlist for ' + slot + ' on ' + date + '? You will be notified if the slot opens up.')) return;
  fetch(SCRIPT_URL,{
    method:'POST',
    body: JSON.stringify({ action:'joinWaitlist', userId:user.uid, userName:user.name,
      email:user.email, instrId, date, slot, occupiedBkId })
  }).then(r=>r.json()).then(res=>{
    if(res.status==='ok'){
      // Store locally so UI reflects waitlisted state
      waitlistData[instrId+'_'+date+'_'+slot] = user.uid;
      localStorage.setItem('limsWaitlist_'+user.uid, JSON.stringify(waitlistData));
      loadSlots();
      toast('Added to waitlist! You will be notified by email if the slot opens.','ok');
    } else { toast(res.msg||'Failed.','err'); }
  });
}
```

### `code.gs` — Add `joinWaitlist` action in `doPost()`:

```js
else if (action === 'joinWaitlist') result = handleJoinWaitlist(data);
```

### `code.gs` — Add handler + Waitlist sheet:

```js
function handleJoinWaitlist(data) {
  if(!data.userId || !data.instrId || !data.date || !data.slot)
    return {status:'error', msg:'Missing fields.'};
  var sh = getSheet('Waitlist');
  // Prevent duplicate
  var existing = sheetToObjects(sh).find(function(r){
    return r.UserID === data.userId && r.InstrID === data.instrId &&
           r.Date === data.date && r.Slot === data.slot && r.Status === 'Waiting';
  });
  if(existing) return {status:'ok', msg:'Already on waitlist.'};
  sh.appendRow([data.userId, data.instrId, data.date, data.slot,
    data.occupiedBkId||'', data.email||'', 'Waiting', nowStr()]);
  return {status:'ok'};
}
```

> Add Waitlist sheet to `setupSheets()`:
```js
var wl = ss.getSheetByName('Waitlist') || ss.insertSheet('Waitlist');
wl.clearContents();
wl.getRange(1,1,1,8).setValues([['UserID','InstrID','Date','Slot','OccupiedBkID','Email','Status','AddedAt']]);
formatHeader(wl, 8);
```

> In `setBookingStatus()` in `code.gs`, when status becomes `'Rejected'` (cancellation), add logic to notify the first waitlisted user:

```js
// After setting status, check waitlist
if(status === 'Rejected'){
  var wlSh = getSheet('Waitlist');
  var wlAll = sheetToObjects(wlSh);
  var bkData = sheetToObjects(sh).find(function(r){ return r.BookingID === bkId; });
  if(bkData){
    var first = wlAll.find(function(r){
      return r.InstrID===bkData.InstrID && r.Date===bkData.Date &&
             r.Slot===bkData.Time && r.Status==='Waiting';
    });
    if(first){
      try {
        MailApp.sendEmail({
          to: first.Email,
          subject: '[AZBT LIMS] Waitlist Alert — Slot Available!',
          body: 'Good news! A slot you were waitlisted for has just opened up.\n\n' +
                'Instrument: ' + bkData.InstrName + '\n' +
                'Date: ' + bkData.Date + '\n' +
                'Time: ' + bkData.Time + '\n\n' +
                'Please log in to AZBT LIMS to book the slot before someone else does.\n\n' +
                '— AZBT LIMS, Loyola College'
        });
      } catch(e) {}
      // Mark this waitlist entry as notified
      var wRow = findRowByField(wlSh, 'Email', first.Email);
      if(wRow > 0){
        var wHm = getHeaderMap(wlSh);
        wlSh.getRange(wRow, wHm['Status']).setValue('Notified');
      }
    }
  }
}
```

---

## 9. QR CODE CHECK-IN — Per-booking QR + instrument QR scan

### Concept:
- Each **booking** gets a unique QR code (encodes booking ID + user ID + date).
- Each **physical instrument** has a printed QR code (encodes its `InstrID`).
- On the day of booking, user must scan the instrument QR first → system validates they are at the correct instrument → session starts.

### `booking.html` — Add QR display in `bookingRow()`:

Add to the actions cell for Approved future bookings:
```js
const isToday = b.date === new Date().toISOString().split('T')[0];
${b.status==='Approved' && isToday
  ? `<button class="btn btn-gold btn-xs" onclick="showQR('${b.id}')" title="Check In via QR">
      <i class="fa-solid fa-qrcode"></i>
     </button>`
  : ''}
```

### `booking.html` — Add QR modal HTML:

```html
<!-- QR Check-In Modal -->
<div class="modal-bg" id="qrModal" onclick="closeModal('qrModal')">
  <div class="modal-box" style="max-width:340px;" onclick="event.stopPropagation()">
    <div class="modal-header">
      <span><i class="fa-solid fa-qrcode" style="color:var(--gold);"></i> Check-In QR Code</span>
      <button class="modal-close" onclick="closeModal('qrModal')">×</button>
    </div>
    <div style="padding:20px;text-align:center;">
      <div id="qrCode" style="margin:0 auto 12px;"></div>
      <div id="qrBkInfo" style="font-size:12px;color:var(--gray);"></div>
      <div style="font-size:12px;color:var(--blue);margin-top:10px;font-weight:600;">
        Scan this with the instrument's QR scanner to start your session.
      </div>
      <div id="qrScanInput" style="margin-top:14px;">
        <input class="form-input" id="instrQrInput" placeholder="Or type/scan instrument QR code here…" style="text-align:center;"/>
        <button class="btn btn-gold btn-sm" style="margin-top:8px;width:100%;" onclick="verifyQrCheckin()">
          <i class="fa-solid fa-play"></i> Start Session
        </button>
      </div>
      <div id="qrMsg" style="margin-top:8px;"></div>
    </div>
  </div>
</div>
```

### `booking.html` — Add QR JS (use free QRCode.js library):

In `<head>`, add:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
```

Functions:
```js
let currentQrBookingId = null;

function showQR(bkId){
  currentQrBookingId = bkId;
  const b = myBookings.find(x => x.id === bkId);
  document.getElementById('qrCode').innerHTML = '';
  // QR encodes: LIMS_CHECKIN|bookingId|userId|instrId|date
  const payload = `LIMS_CHECKIN|${bkId}|${user.uid}|${b.instrId}|${b.date}`;
  new QRCode(document.getElementById('qrCode'), {
    text: payload, width:200, height:200,
    colorDark:'#1E3057', colorLight:'#fff'
  });
  document.getElementById('qrBkInfo').innerHTML =
    `Booking: <strong>${bkId}</strong> · ${b.instrName} · ${b.date} ${formatTime(b.time)}`;
  document.getElementById('instrQrInput').value = '';
  setMsg('qrMsg','');
  openModal('qrModal');
}

function verifyQrCheckin(){
  const scanned = document.getElementById('instrQrInput').value.trim();
  const b = myBookings.find(x => x.id === currentQrBookingId);
  if(!b){ setMsg('qrMsg','Booking not found.'); return; }
  // The physical instrument QR encodes its InstrID (e.g. "INSTR|I001")
  const instrIdFromQr = scanned.replace('INSTR|','').toUpperCase();
  if(instrIdFromQr !== b.instrId.toUpperCase()){
    setMsg('qrMsg','⚠ Wrong instrument! This QR belongs to a different instrument.');
    return;
  }
  // Check in
  fetch(SCRIPT_URL,{
    method:'POST',
    body: JSON.stringify({ action:'checkInBooking', bookingId: currentQrBookingId, userId: user.uid })
  }).then(r=>r.json()).then(res=>{
    if(res.status==='ok'){
      closeModal('qrModal');
      toast('Session started! Timer is running.','ok');
      fetchData();
      startProgressTimer(b);
    } else { setMsg('qrMsg', res.msg||'Check-in failed.'); }
  });
}
```

### `code.gs` — Add `checkInBooking` action and handler:

```js
// In doPost():
else if (action === 'checkInBooking') result = handleCheckIn(data);

// Handler:
function handleCheckIn(data){
  var sh = getSheet(SH_BOOKINGS);
  var row = findRowByField(sh, 'BookingID', data.bookingId);
  if(row < 0) return {status:'error', msg:'Booking not found.'};
  var hm = getHeaderMap(sh);
  // Add a CheckedInAt column (add to sheet manually or via setupSheets)
  sh.getRange(row, hm['CheckedInAt']).setValue(nowStr());
  sh.getRange(row, hm['Status']).setValue('Active');
  return {status:'ok'};
}
```

> Add `CheckedInAt` column to the Bookings sheet header row manually.  
> Update `bookingToObj()` to include `checkedInAt: b.CheckedInAt||''`.

### Print instrument QR codes (for sticking on instruments):

In `admin.html` instruments panel, add a button per instrument row:
```js
`<button class="btn btn-gold btn-xs" onclick="printInstrQR('${i.id}','${i.name}')" title="Print QR">
  <i class="fa-solid fa-qrcode"></i>
</button>`
```

```js
function printInstrQR(instrId, instrName){
  const w = window.open('','_blank');
  w.document.write(`<!DOCTYPE html><html><head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"><\/script>
    </head><body style="text-align:center;font-family:Arial;padding:40px;">
    <h3>${instrName}</h3><p style="font-size:11px;color:#555;">${instrId}</p>
    <div id="iqr" style="display:inline-block;"></div>
    <p style="font-size:10px;color:#888;margin-top:12px;">Scan to check in · AZBT LIMS</p>
    <script>
      new QRCode(document.getElementById('iqr'),{text:'INSTR|${instrId}',width:200,height:200});
      setTimeout(()=>window.print(),800);
    <\/script></body></html>`);
  w.document.close();
}
```

---

## 10. PROGRESS TIMER — Time remaining for active booking

### `booking.html` — Add progress bar HTML in the profile/dashboard panel:

After the stats grid, add:
```html
<div id="activeSessionBar" style="display:none;margin-top:16px;background:linear-gradient(135deg,#1E3057,#2a4070);border-radius:12px;padding:14px 16px;color:#fff;">
  <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;">
    <span style="font-weight:700;font-size:13px;"><i class="fa-solid fa-circle-play" style="color:#4ade80;"></i> Active Session</span>
    <span id="activeSessionName" style="font-size:12px;color:var(--gold2);"></span>
  </div>
  <div style="background:rgba(255,255,255,.15);border-radius:6px;height:10px;overflow:hidden;">
    <div id="sessionProgressBar" style="height:100%;background:linear-gradient(to right,#4ade80,#22c55e);transition:width .5s;width:0%;border-radius:6px;"></div>
  </div>
  <div style="display:flex;justify-content:space-between;margin-top:6px;font-size:11px;color:rgba(255,255,255,.7);">
    <span id="sessionTimeElapsed">0:00 elapsed</span>
    <span id="sessionTimeRemaining">—</span>
  </div>
</div>
```

### `booking.html` — Add timer JS:

```js
let sessionTimer = null;

function startProgressTimer(booking){
  clearInterval(sessionTimer);
  const checkedIn = new Date(booking.checkedInAt);
  const totalMs   = booking.dur * 60 * 60 * 1000;
  const bar       = document.getElementById('activeSessionBar');
  bar.style.display = 'block';
  document.getElementById('activeSessionName').textContent = booking.instrName;

  sessionTimer = setInterval(()=>{
    const elapsed = Date.now() - checkedIn.getTime();
    const pct     = Math.min(100, (elapsed / totalMs) * 100);
    const rem     = Math.max(0, totalMs - elapsed);
    document.getElementById('sessionProgressBar').style.width = pct + '%';
    document.getElementById('sessionTimeElapsed').textContent  = msToHM(elapsed) + ' elapsed';
    document.getElementById('sessionTimeRemaining').textContent = 'Remaining: ' + msToHM(rem);
    if(rem === 0){ clearInterval(sessionTimer); bar.style.backgroundColor='#991b1b'; }
  }, 10000);
}

function msToHM(ms){
  const m = Math.floor(ms/60000);
  return `${Math.floor(m/60)}h ${m%60}m`;
}
```

> In `initProfile()`, check for any `Active` booking today and auto-start the timer:
```js
const active = myBookings.find(b => b.status==='Active' &&
  b.date === new Date().toISOString().split('T')[0] && b.checkedInAt);
if(active) startProgressTimer(active);
```

---

## 11. HOD INSTRUMENT RECOMMENDATION

### `booking.html` — Add a nav tab (or a floating button) for recommendations:

Add tab button in nav:
```html
<button class="nav-tab" onclick="switchPanel('recommend')">
  <i class="fa-solid fa-lightbulb"></i> Recommend
</button>
```

Add panel HTML:
```html
<div id="recommendPanel" class="panel">
  <div style="font-family:'Playfair Display',serif;font-size:17px;color:var(--blue);margin-bottom:16px;display:flex;align-items:center;gap:8px;">
    <i class="fa-solid fa-lightbulb" style="color:var(--gold);"></i> Recommend an Instrument to HOD
  </div>
  <div class="form-group">
    <label class="form-label"><i class="fa-solid fa-microscope"></i> Instrument Name / Type <span class="req">*</span></label>
    <input class="form-input" id="recName" placeholder="e.g. Flow Cytometer, HPLC, etc."/>
  </div>
  <div class="form-group">
    <label class="form-label"><i class="fa-solid fa-building"></i> Suggested Manufacturer / Model</label>
    <input class="form-input" id="recModel" placeholder="e.g. BD FACSLyric, Agilent 1260…"/>
  </div>
  <div class="form-group">
    <label class="form-label"><i class="fa-solid fa-flask"></i> Research Application / Use Case <span class="req">*</span></label>
    <textarea class="form-input" id="recUse" rows="3" placeholder="Describe what experiments or research this instrument would enable…" style="resize:vertical;"></textarea>
  </div>
  <div class="form-group">
    <label class="form-label"><i class="fa-solid fa-comment"></i> Additional Notes</label>
    <textarea class="form-input" id="recNotes" rows="2" style="resize:vertical;" placeholder="Priority, estimated cost range, references, etc."></textarea>
  </div>
  <div id="recMsg"></div>
  <button class="btn btn-gold" onclick="submitRecommendation()" style="margin-top:6px;">
    <i class="fa-solid fa-paper-plane"></i> Submit Recommendation
  </button>
</div>
```

Add `'recommend'` to the panel arrays in `switchPanel()`.

### `booking.html` — Add `submitRecommendation()` JS:

```js
function submitRecommendation(){
  const name  = document.getElementById('recName').value.trim();
  const model = document.getElementById('recModel').value.trim();
  const use   = document.getElementById('recUse').value.trim();
  const notes = document.getElementById('recNotes').value.trim();
  if(!name||!use){ setMsg('recMsg','Please fill in the required fields.'); return; }
  fetch(SCRIPT_URL,{
    method:'POST',
    body: JSON.stringify({ action:'submitRecommendation', userId:user.uid,
      userName:user.name, instrName:name, model, useCase:use, notes })
  }).then(r=>r.json()).then(res=>{
    if(res.status==='ok'){
      setMsg('recMsg','Recommendation submitted to HOD successfully!','success');
      document.getElementById('recName').value='';
      document.getElementById('recModel').value='';
      document.getElementById('recUse').value='';
      document.getElementById('recNotes').value='';
    } else { setMsg('recMsg', res.msg||'Failed.'); }
  });
}
```

### `code.gs` — Add action and handler:

```js
// In doPost():
else if (action === 'submitRecommendation') result = handleRecommendation(data);

// Handler:
function handleRecommendation(data){
  if(!data.instrName||!data.useCase) return {status:'error',msg:'Missing fields.'};
  var sh = getSheet('Recommendations');
  sh.appendRow([data.userId||'', data.userName||'', data.instrName,
    data.model||'', data.useCase, data.notes||'', nowStr()]);
  try {
    MailApp.sendEmail({
      to: ADMIN_EMAIL,
      subject: '[AZBT LIMS] New Instrument Recommendation from ' + (data.userName||data.userId),
      body: 'A user has recommended a new instrument for the department.\n\n' +
            'Recommended by: ' + (data.userName||data.userId) + '\n' +
            'Instrument: ' + data.instrName + '\n' +
            'Model: ' + (data.model||'—') + '\n' +
            'Use Case: ' + data.useCase + '\n' +
            'Notes: ' + (data.notes||'—') + '\n\n' +
            'Submitted: ' + nowStr()
    });
  } catch(e) {}
  return {status:'ok'};
}
```

> Add `Recommendations` sheet to `setupSheets()` with headers: `UserID, UserName, InstrName, Model, UseCase, Notes, SubmittedAt`.

---

## 12. SUMMARY OF ALL `code.gs` ADDITIONS

| New action string | New function | New sheet needed |
|---|---|---|
| `reportIssue` | `handleReportIssue()` | `IssueReports` |
| `joinWaitlist` | `handleJoinWaitlist()` | `Waitlist` |
| `checkInBooking` | `handleCheckIn()` | — (add `CheckedInAt` col to Bookings) |
| `submitRecommendation` | `handleRecommendation()` | `Recommendations` |
| `getIssueReports` | `handleGetIssueReports()` (mirror getAllBookings) | — |

Add all 4 new sheets in `setupSheets()`.  
Add `CheckedInAt` and `AdminNote` (if missing) columns to the Bookings sheet.

---

## 13. QUICK CHECKLIST — Where each change goes

| Feature | booking.html | admin.html | code.gs |
|---|---|---|---|
| Slot shows "Yours" / "Booked" correctly | `loadSlots()` rewrite | — | — |
| Multi-hour overlap fix | `loadSlots()` | `loadSlots()` equivalent (manual booking modal) | Conflict check already in `handleSubmitBooking` |
| User cancel booking | `cancelSlotBooking()`, `bookingRow()` | — | `handleCancelBooking` already exists |
| 12h AM/PM time | Add `formatTime()`, apply everywhere | Add `formatTime()`, apply everywhere | — |
| Calendar race condition fix | `fetchData()` → `Promise.all` | `fetchAll()` → `Promise.all` | — |
| Calendar day popup | `calClick()`, add modal HTML | `showDayModal()` already exists | — |
| Per-booking print (user) | `printSingleBooking()`, `bookingRow()` | — | — |
| Dual logo in print | `printMyBookings()` | `printHeader()` | — |
| Instrument issue report | Report modal + `submitReport()` | Add Issues panel | `handleReportIssue()`, IssueReports sheet |
| Notification bell | Bell HTML + `checkNotifications()` | Same pattern for admin | — |
| Waitlist | `joinWaitlist()` in `loadSlots()` | — | `handleJoinWaitlist()`, Waitlist sheet, notify on cancel |
| QR Check-in | QR modal + `showQR()`, `verifyQrCheckin()` | `printInstrQR()` in instruments panel | `handleCheckIn()`, add CheckedInAt col |
| Progress timer | `activeSessionBar` HTML + `startProgressTimer()` | — | — |
| HOD recommendation | Recommend panel + `submitRecommendation()` | View recommendations panel | `handleRecommendation()`, Recommendations sheet |
