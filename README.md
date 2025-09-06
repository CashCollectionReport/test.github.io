<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Cash Collection Report</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <!-- jsPDF for client-side PDF generation -->
  <script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
  <!-- EmailJS (optional, for emailing with attachment) -->
  <script src="https://cdn.jsdelivr.net/npm/@emailjs/browser@3/dist/email.min.js"></script>
  <style>
    :root {font-family: system-ui, Segoe UI, Roboto, Arial, sans-serif;}
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body { margin: 0; background:#f7fbf7; }

    /* Container/card frame */
    .container { max-width: min(1100px, 96vw); margin: 0 auto; padding: 16px; }
    .card { background: #252e97; border-radius: 18px; box-shadow: 0 10px 30px rgba(0,0,0,.08); overflow: hidden; }
    .header { padding: 20px 24px; border-bottom: 1px solid rgba(255,255,255,.15); color:#fff; }
    h1 { margin: 0 0 4px; font-size: clamp(18px, 2.5vw + 10px, 24px); }
    p.lead { margin: 0; color: #f0f3ff; opacity:.85; }

    /* Layout: fixed-width sidebar + scrollable content */
    .layout {
      display: grid;
      grid-template-columns: 260px 1fr;
      gap: 0;
      min-height: calc(100vh - 140px); /* viewport height minus header + margins */
      background:#252e97;
    }

    /* Sticky sidebar – stays put while main content scrolls */
    .sidebar {
      background: #fff;
      padding: 12px;
      display:flex; flex-direction:column; gap:10px;
      position: sticky; top: 0; align-self: start;
      max-height: calc(100vh - 140px);
      overflow: auto;
      border-right: 1px solid #e5e7eb;
    }

    /* Main content scrolls independently of the sidebar */
    .content {
      background:#fff;
      padding: 16px;
      overflow: auto;
      max-height: calc(100vh - 140px);
    }

    /* Buttons in the sidebar */
    .navbtn {
      text-align: left; border: 1px solid rgba(0,0,0,.12); background: transparent; color:#0e0e0e;
      padding: 10px 12px; border-radius: 14px; cursor: pointer; font-weight:600;
      display:flex; align-items:flex-start; justify-content:space-between; gap:8px;
      transition: transform .05s ease, background .2s ease, border-color .2s ease;
      white-space: normal; /* allow text wrap */
      line-height: 1.2;
    }
    .navbtn small { opacity:.8; font-weight:500; display:block; }
    .navbtn:active { transform: translateY(1px); }
    .navbtn:disabled { opacity:.5; cursor:not-allowed; }
    .navbtn[data-theme="p1"] { border-color:#60a5fa; }
    .navbtn[data-theme="p2"] { border-color:#c084fc; }
    .navbtn[data-theme="p3"] { border-color:#5eead4; }
    .navbtn[data-theme="p4"] { border-color:#86efac; }
    .navbtn.active[data-theme="p1"] { background:#2563eb; border-color:#2563eb; color:#fff; }
    .navbtn.active[data-theme="p2"] { background:#7c3aed; border-color:#7c3aed; color:#fff; }
    .navbtn.active[data-theme="p3"] { background:#0d9488; border-color:#0d9488; color:#fff; }
    .navbtn.active[data-theme="p4"] { background:#16a34a; border-color:#16a34a; color:#fff; }
    .check { font-size: 12px; color:#052e16; padding:2px 6px; border-radius:999px; display:none; }
    .navbtn.done .check { display:inline-block; }
    .navbtn.done[data-theme="p1"] .check { background:#60a5fa; }
    .navbtn.done[data-theme="p2"] .check { background:#c084fc; }
    .navbtn.done[data-theme="p3"] .check { background:#5eead4; color:#064e3b; }
    .navbtn.done[data-theme="p4"] .check { background:#86efac; }

    /* Grid helpers */
    .grid { display: grid; gap: 16px; grid-template-columns: repeat(12,minmax(0,1fr)); }
    .col-3 { grid-column: span 3; } .col-4 { grid-column: span 4; } .col-6 { grid-column: span 6; } .col-8 { grid-column: span 8; } .col-12 { grid-column: span 12; }
    @media (max-width: 960px) { .col-3, .col-4, .col-6, .col-8 { grid-column: span 12; } }

    label { display:block; font-weight:600; margin-bottom:6px; color:#0a0a0a; }
    select, input[type="text"], input[type="date"], input[type="email"], textarea, input[type="file"], input[type="number"] {
      width: 100%; padding: 10px 12px; border-radius: 10px; border: 1px solid #d6d6de; background:#fff; min-width: 0;
    }
    textarea { min-height: 96px; resize: vertical; }
    .muted { color:#475569; font-size: 12px; }
    .row { display:flex; gap: 10px; align-items:center; flex-wrap: wrap; }
    .required::after { content:" *"; color:#ef4444; }

    .out {
      background:#0f172a; color:#e5e7eb; border-radius: 12px; padding: 14px; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
      display:flex; justify-content: space-between; align-items: center; gap: 10px; word-break: break-word; flex-wrap: wrap;
    }

    .btn { border:0; padding: 10px 14px; border-radius: 12px; cursor:pointer; color:#fff; font-weight:600; transition: transform .05s ease, filter .15s ease; }
    .btn:active { transform: translateY(1px); }
    .btn-blue{background:#2563eb;} .btn-green{background:#16a34a;} .btn-purple{background:#7c3aed;} .btn-teal{background:#0d9488;}
    .btn-indigo{background:#4338ca;} .btn-rose{background:#e11d48;} .btn-amber{background:#f59e0b;color:#111827;} .btn-slate{background:#64748b;}
    .btn:hover { filter: brightness(1.05); }
    .btn[disabled] { opacity: .5; cursor: not-allowed; }

    .footer { display:flex; justify-content:space-between; align-items:center; gap:12px; padding:12px 16px; border-top:1px solid #e5e7eb; background:#fff; flex-wrap: wrap; }

    .panel { display:none; } .panel.active { display:block; }
    .chip { display:inline-block; padding:4px 8px; background:#eef2ff; border-radius: 999px; font-size: 12px; }

    .dropzone { border: 2px dashed #94a3b8; border-radius: 12px; padding: 18px; text-align:center; background:#f8fafc; transition: background .2s, border-color .2s; }
    .dropzone.dragover { background:#eff6ff; border-color:#3b82f6; }
    .files { margin-top:12px; border:1px solid #e5e7eb; border-radius:12px; padding:8px; max-height: 220px; overflow:auto; background:#fff; }
    .file-row { display:flex; justify-content:space-between; align-items:center; gap:8px; padding:6px 8px; border-bottom:1px dashed #e5e7eb; }
    .file-row:last-child { border-bottom:0; }
    .file-meta { font-size: 12px; color:#475569; }
    .file-actions { display:flex; gap:8px; }

    .table { width: 100%; border-collapse: collapse; background:#fff; border-radius: 12px; overflow: hidden; border:1px solid #e5e7eb; table-layout: fixed; }
    .table th, .table td { padding:10px; border-bottom:1px solid #e5e7eb; text-align:left; word-break: break-word; }
    .table th { background:#f8fafc; font-weight:700; }
    .table tfoot td { font-weight:700; background:#f9fafb; }
    .table-wrap { overflow:auto; }

    .num { width: 140px; }
    .num input { width: 100%; padding:8px 10px; border-radius:8px; border:1px solid #d6d6de; text-align:right; }

    .kpi { display:flex; gap:10px; flex-wrap:wrap; }
    .kpi > div { background:#f8fafc; border:1px solid #e5e7eb; border-radius:12px; padding:10px 12px; min-width:180px; }
    .status { display:inline-block; padding:4px 8px; border-radius:999px; font-size:12px; font-weight:700; }
    .status.ok { background:#dcfce7; color:#14532d; }
    .status.bad { background:#fee2e2; color:#7f1d1d; }

    .help { font-size:12px; color:#475569; }
    .subsection { margin-top:16px; }
    .subsection h4 { margin: 0 0 8px 0; }

    .toast { position: fixed; right: 16px; bottom: 16px; background: #111827; color:#fff;
      padding: 10px 14px; border-radius: 10px; box-shadow: 0 6px 24px rgba(0,0,0,.18);
      opacity: 0; transform: translateY(10px); transition: .2s ease;
      z-index: 9999; pointer-events: none; }
    .toast.show { opacity: 1; transform: translateY(0); }

    /* Mobile tweaks: sidebar becomes horizontal scroller */
    @media (max-width: 1024px) {
      .layout { grid-template-columns: 1fr; }
      .sidebar {
        position: static; max-height: none; overflow: auto;
        display: flex; flex-direction: row; gap: 8px; white-space: nowrap;
        border-right: 0; border-bottom: 1px solid #e5e7eb;
      }
      .navbtn { flex: 0 0 auto; }
      .content { max-height: none; }
    }

    /* Very small screens: tighten cells */
    @media (max-width: 640px) {
      .num { width: 120px; }
      .table th, .table td { padding: 8px; }
    }
  </style>

  <script>
    // If you previously had a mismatched variable, make sure captureBtn is defined:
    (function () {
      if (!window.captureBtn) {
        const btn = document.getElementById("captureBtn");
        if (btn) window.captureBtn = btn;
      }
    })();
  </script>

</head>
<body>
  <div class="container">
    <div class="card">
      <div class="header">
        <h1>Cash Collection Report</h1>
        <p class="lead">Declare your OTP End of Shift (EOS) report. Note: Combine EOS if multiple reports are generated within the same shift.</p>
      </div>

      <div class="layout">
        <!-- Left sidebar -->
        <aside class="sidebar" role="tablist" aria-orientation="vertical">
          <button class="navbtn active" id="btn-page1" data-target="page1" data-theme="p1" role="tab" aria-selected="true">
            1. Details <small>(Staff/Line/Station/...)</small><span class="check">✓</span>
          </button>
          <button class="navbtn" id="btn-page2" data-target="page2" data-theme="p2" role="tab" aria-selected="false" disabled>
            2. Sale Details <small>(Sale Summary)</small><span class="check">✓</span>
          </button>
          <button class="navbtn" id="btn-page3" data-target="page3" data-theme="p3" role="tab" aria-selected="false" disabled>
            3. Uploads <small>(PDF/Images/Camera)</small><span class="check">✓</span>
          </button>
          <button class="navbtn" id="btn-page4" data-target="page4" data-theme="p4" role="tab" aria-selected="false" disabled>
            4. Review & Submit <small>(PDF/Email)</small><span class="check">✓</span>
          </button>
        </aside>

        <!-- Main content -->
        <main class="content">
          <!-- PAGE 1 -->
          <section class="panel active" id="panel-page1" data-section="page1">
            <div class="grid">
              <div class="col-6">
                <label class="required" for="staffSearch">Staff (type to search)</label>
                <input id="staffSearch" type="text" placeholder="Search staff by name or code..." />
                <div class="muted">Start typing; the list filters automatically.</div>
              </div>
              <div class="col-6">
                <label class="required" for="staffName">Staff List</label>
                <select id="staffName">
                  <option value="" selected disabled>— Choose staff —</option>
                </select>
              </div>

              <div class="col-6">
                <label class="required" for="line">Line</label>
                <select id="line">
                  <option value="" selected disabled>— Choose line —</option>
                </select>
                <div class="muted">The code in parentheses is used in the filename.</div>
              </div>

              <div class="col-6">
                <label class="required" for="station">Station</label>
                <select id="station">
                  <option value="" selected disabled>— Choose station —</option>
                </select>
              </div>

              <div class="col-6">
                <label class="required" for="equip">Equipment (by Station)</label>
                <div class="row">
                  <select id="equip" style="flex:1">
                    <option value="" selected disabled>— Choose equipment —</option>
                  </select>
                  <span class="chip" id="equipPreview">—</span>
                </div>
                <div class="muted">Only shows equipment IDs at the selected station.</div>
              </div>

              <div class="col-3">
                <label class="required" for="shift">Shift</label>
                <select id="shift">
                  <option value="" selected disabled>— Choose shift —</option>
                </select>
              </div>

              <div class="col-3">
                <label class="required" for="cdm">CDM Location</label>
                <select id="cdm">
                  <option value="" selected disabled>— Choose CDM —</option>
                </select>
              </div>

              <div class="col-4">
                <label class="required" for="theDate">Date</label>
                <input id="theDate" type="date" />
                <div class="muted">Format: YYYYMMDD.</div>
              </div>
            </div>
          </section>

          <!-- PAGE 2 -->
          <section class="panel" id="panel-page2" data-section="page2">
            <div class="grid">
              <div class="col-12">
                <label class="required" for="shiftNo">OTP Shift No</label>
                <input id="shiftNo" type="text" placeholder="e.g., 000xxxx" />
                <div class="muted">Will be appended to the filename. (Required to proceed.)</div>
              </div>

              <!-- A. OTP Cash Denomination -->
              <div class="col-12">
                <h3 style="margin:18px 0 8px;">A. OTP Cash Denomination (Notes & Coins)</h3>
                <div class="help">Enter quantity per denomination. Amount auto-calculates.</div>
                <div class="table-wrap">
                  <table class="table" id="cashDenomTable" aria-label="OTP Cash Denomination">
                    <thead>
                      <tr>
                        <th>Denomination</th>
                        <th class="num">Qty</th>
                        <th class="num">Amount (RM)</th>
                      </tr>
                    </thead>
                    <tbody>
                      <!-- Notes -->
                      <tr data-denom="100"><td>RM100</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="50"><td>RM50</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="20"><td>RM20</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="10"><td>RM10</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="5"><td>RM5</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="1"><td>RM1</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <!-- Coins -->
                      <tr data-denom="0.50"><td>50 sen</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="0.20"><td>20 sen</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="0.10"><td>10 sen</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                      <tr data-denom="0.05"><td>5 sen</td><td class="num"><input type="number" min="0" step="1" value="0" /></td><td class="num"><input type="number" readonly /></td></tr>
                    </tbody>
                    <tfoot>
                      <tr>
                        <td style="text-align:right">Total Cash Counted</td>
                        <td class="num"><input id="cashDenomTotalQty" type="number" readonly /></td>
                        <td class="num"><input id="cashDenomTotalAmt" type="number" readonly /></td>
                      </tr>
                    </tfoot>
                  </table>
                </div>
              </div>

              <!-- B. Sales by Payment Method & Product Type (TNG + Refund) -->
              <div class="col-12">
                <h3 style="margin:18px 0 8px;">B. TNG & Refund Matrix</h3>
                <div class="help">Enter RM amounts. Refunds as positive numbers (system treats as outflow).</div>
                <div class="table-wrap">
                  <table class="table" id="salesMatrix" aria-label="Sales Matrix">
                    <thead>
                      <tr>
                        <th>Product Type ↓ / Payment →</th>
                        <th>Cash</th>
                        <th>Credit Card</th>
                        <th>Debit Card</th>
                        <th>Spay</th>
                        <th>Other E-Wallet</th>
                        <th>DuitNow</th>
                        <th class="num">Row Total</th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr data-row="tng_card_sale">
                        <td>TNG Card Sale</td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td class="num"><input type="number" readonly /></td>
                      </tr>
                      <tr data-row="tng_card_sale_reload">
                        <td>TNG Card Sale + Reload</td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td class="num"><input type="number" readonly /></td>
                      </tr>
                      <tr data-row="tng_reload_only">
                        <td>TNG Reload Only</td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td class="num"><input type="number" readonly /></td>
                      </tr>
                      <tr data-row="refund">
                        <td>Refund</td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td><input type="number" min="0" step="0.01" value="0" /></td>
                        <td class="num"><input type="number" readonly /></td>
                      </tr>
                    </tbody>
                    <tfoot>
                      <tr>
                        <td style="text-align:right">Column Totals</td>
                        <td class="num"><input id="tot_cash" type="number" readonly /></td>
                        <td class="num"><input id="tot_cc" type="number" readonly /></td>
                        <td class="num"><input id="tot_dc" type="number" readonly /></td>
                        <td class="num"><input id="tot_spay" type="number" readonly /></td>
                        <td class="num"><input id="tot_ewallet" type="number" readonly /></td>
                        <td class="num"><input id="tot_duitnow" type="number" readonly /></td>
                        <td class="num"><input id="grandTotal" type="number" readonly /></td>
                      </tr>
                    </tfoot>
                  </table>
                </div>
              </div>

              <!-- C. Per-Product Subtables -->
              <div class="col-12">
                <h3 style="margin:18px 0 8px;">C. Per-Product Sales</h3>
                <div class="help">Enter RM amounts for each product by payment method. These feed into totals & reconciliation.</div>

                <div id="productSubtables"></div>

                <div class="kpi" style="margin-top:10px">
                  <div>
                    <div class="muted">Cash from Denominations</div>
                    <div style="font-weight:700">RM <span id="kpiCashDenom">0.00</span></div>
                  </div>
                  <div>
                    <div class="muted">Cash from All Sales (after Refunds)</div>
                    <div style="font-weight:700">RM <span id="kpiCashMatrix">0.00</span></div>
                  </div>
                  <div>
                    <div class="muted">Reconciliation</div>
                    <div><span id="kpiRecon" class="status">Calculating…</span></div>
                  </div>
                </div>
              </div>
            </div>
          </section>

          <!-- PAGE 3 (Uploads) -->
          <section class="panel" id="panel-page3" data-section="page3">
            <div class="grid">
              <div class="col-12">
                <label class="required">Upload Files (PDF / Images / Camera)</label>
                <div id="dropzone" class="dropzone">
                  <p class="muted">Drag & drop files here, or use the buttons below.</p>
                  <input id="fileInput" type="file" multiple accept="image/*,application/pdf" style="display:none" />
                  <input id="cameraInput" type="file" accept="image/*" capture="environment" style="display:none" />
                  <div class="row" style="justify-content:center; margin-top:8px">
                    <button id="chooseFilesBtn" class="btn btn-indigo" type="button">Choose Files</button>
                    <button id="captureBtn" class="btn btn-teal" type="button">Capture from Camera</button>
                  </div>
                </div>
                <div id="filesList" class="files" aria-live="polite"></div>
                <div class="muted" id="filesSummary">No files added yet.</div>
              </div>
            </div>
          </section>

          <!-- PAGE 4 (Review & Submit) -->
          <section class="panel" id="panel-page4" data-section="page4">
            <div class="grid">
              <div class="col-12">
                <label>Generated Filename</label>
                <div class="out">
                  <span id="filename">—</span>
                  <div class="right" style="display:flex; gap:8px; flex-wrap: wrap;">
                    <button class="btn btn-purple inline" id="copyBtn" type="button">Copy Name</button>
                    <button class="btn btn-amber inline" id="resetBtn" type="button">Reset</button>
                  </div>
                </div>
                <div class="muted">Submit will build a PDF report using this name and download it.</div>
              </div>

              <!-- Email form -->
              <div class="col-12">
                <h3 style="margin:18px 0 8px;">Email Submission</h3>
                <div class="grid">
                  <div class="col-6">
                    <label class="required" for="emailTo">Send To (email)</label>
                    <input id="emailTo" type="email" placeholder="cs@yourdomain.com" />
                  </div>
                  <div class="col-6">
                    <label for="emailCc">CC</label>
                    <input id="emailCc" type="email" placeholder="supervisor@yourdomain.com" />
                  </div>
                  <div class="col-12">
                    <label for="emailSubject">Subject</label>
                    <input id="emailSubject" type="text" placeholder="Cash Collection Report" />
                  </div>
                  <div class="col-12">
                    <label for="emailMessage">Message</label>
                    <textarea id="emailMessage" placeholder="Please find the attached Cash Collection Report."></textarea>
                  </div>
                  <div class="col-12 row" style="gap:8px">
                    <button class="btn btn-green" id="sendEmailBtn" type="button" disabled>Send Email (attach PDF)</button>
                    <button class="btn btn-blue" id="downloadPdfBtn" type="button">Download PDF Only</button>
                  </div>
                  <div class="col-12">
                    <div class="muted">Tip: Configure EmailJS (keys below) for automatic emailing with attachment. Without it, a mail window opens (no auto-attach) and the PDF downloads for manual attaching.</div>
                  </div>
                </div>
              </div>
            </div>
          </section>

          <!-- Footer Nav -->
          <div class="footer">
            <button class="btn btn-slate" id="backBtn" disabled>← Back</button>
            <div class="proceed">
              <button class="btn btn-blue" id="nextBtn" disabled>Next →</button>
              <button class="btn btn-green" id="submitBtn" style="display:none" type="button">Submit (Generate PDF)</button>
            </div>
          </div>
        </main>
      </div>
    </div>
  </div>

  <div id="toast" class="toast" role="status" aria-live="polite">Working…</div>

  <script>
    /* ========= CONFIG — EmailJS (optional, for emailing attachments) =========
       1) Create an EmailJS account → add a Service and a Template.
       2) Put your Public Key, Service ID and Template ID below.
       3) In the template, add fields: to_email, cc_email, email_subject, email_message.
       4) Enable attachments in the template and map: attachment (base64), attachment_filename.
    */
    const EMAILJS_PUBLIC_KEY  = "";         // e.g. "HkxxxxXX_xxxxxxxxx"
    const EMAILJS_SERVICE_ID  = "";         // e.g. "service_xxxxxx"
    const EMAILJS_TEMPLATE_ID = "";         // e.g. "template_xxxxxx"
    if (EMAILJS_PUBLIC_KEY) { try { emailjs.init(EMAILJS_PUBLIC_KEY); } catch(e){} }

    /* === Data === */
    const LINES = [
      { label: "Blue",  code: "BL" },
      { label: "Red",   code: "RD" },
      { label: "Green", code: "GN" },
    ];
    const STATIONS = [
      { label: "Universiti Station", code: "SM01", line: "BL"},
      { label: "Melaban", code: "SM02", line: "BL"},
      { label: "Sigitin", code: "SM03", line: "BL"},
      { label: "Unimas", code: "SM04", line: "BL"},
      { label: "Heart Centre", code: "SM05", line: "BL"},
      { label: "Riveria", code: "SM06", line: "BL"},
      { label: "Stutong", code: "SM07", line: "BL" },
      { label: "Wan Alwi", code: "SM08", line: "BL" },
      { label: "Viva City Mall", code: "SM09", line: "BL" },
      { label: "The Spring", code: "SM11", line: "BL" },
      { label: "Batu Lintang", code: "SM12", line: "BL" },
      { label: "Sarawak General Hospital", code: "SM13", line: "BL" },
      { label: "Hikmah Exchange", code: "SM14", line: "BL" },
      { label: "Kuching Sentral", code: "SR5", line: "RD" },
      { label: "Kuching International Airport", code: "SR6", line: "RD" },
      { label: "Pelita Height", code: "SR7", line: "RD" },
      { label: "Tun Jugah", code: "SR8", line: "RD" },
      { label: "Swimburne", code: "SR9", line: "RD" },
      { label: "Simpang Tiga", code: "SR10/SM10", line: "RD"},
      { label: "Tun Razak", code: "SR11", line: "RD" },
      { label: "Pending", code: "SR12/DM01", line: "GN" },
      { label: "Wisma Bapa", code: "DM03", line: "GN" },
      { label: "Menara Pelita", code: "DM04", line: "GN" },
      { label: "Stadium", code: "DM05", line: "GN" },
      { label: "Bandar Baru Samariang", code: "DM07", line: "GN" },
      { label: "Sungai Batu", code: "DM08", line: "GN" },
      { label: "Damai Sentral", code: "DM10", line: "GN" },
    ];
    const EQUIPMENTS = [
      { id: "SM01-AFC-OTP-00001", station: "SM01" }, { id: "SM01-AFC-OTP-00002", station: "SM01" },
      { id: "SM02-AFC-OTP-00001", station: "SM02" }, { id: "SM02-AFC-OTP-00002", station: "SM02" },
      { id: "SM03-AFC-OTP-00001", station: "SM03" }, { id: "SM03-AFC-OTP-00002", station: "SM03" },
      { id: "SM04-AFC-OTP-00001", station: "SM04" }, { id: "SM04-AFC-OTP-00002", station: "SM04" },
      { id: "SM05-AFC-OTP-00001", station: "SM05" }, { id: "SM05-AFC-OTP-00002", station: "SM05" },
      { id: "SM06-AFC-OTP-00001", station: "SM06" }, { id: "SM06-AFC-OTP-00002", station: "SM06" },
      { id: "SM07-AFC-OTP-00001", station: "SM07" }, { id: "SM07-AFC-OTP-00002", station: "SM07" },
      { id: "SM08-AFC-OTP-00001", station: "SM08" }, { id: "SM08-AFC-OTP-00002", station: "SM08" },
      { id: "SM09-AFC-OTP-00001", station: "SM09" }, { id: "SM09-AFC-OTP-00002", station: "SM09" },
      { id: "SR10-AFC-OTP-00001", station: "SR10/SM10" }, { id: "SR10-AFC-OTP-00002", station: "SR10/SM10" },
      { id: "SM11-AFC-OTP-00001", station: "SM11" }, { id: "SM11-AFC-OTP-00002", station: "SM11" },
      { id: "SM12-AFC-OTP-00001", station: "SM12" }, { id: "SM12-AFC-OTP-00002", station: "SM12" },
      { id: "SM13-AFC-OTP-00001", station: "SM13" }, { id: "SM13-AFC-OTP-00002", station: "SM13" },
      { id: "SM14-AFC-OTP-00001", station: "SM14" }, { id: "SM14-AFC-OTP-00002", station: "SM14" },
      { id: "SR5-AFC-OTP-00001", station: "SR5" }, { id: "SR5-AFC-OTP-00002", station: "SR5" }, { id: "SR5-AFC-OTP-00003", station: "SR5" }, { id: "SR5-AFC-OTP-00004", station: "SR5" },
      { id: "SR6-AFC-OTP-00001", station: "SR6" }, { id: "SR6-AFC-OTP-00002", station: "SR6" },
      { id: "SR7-AFC-OTP-00001", station: "SR7" }, { id: "SR7-AFC-OTP-00002", station: "SR7" },
      { id: "SR8-AFC-OTP-00001", station: "SR8" }, { id: "SR8-AFC-OTP-00002", station: "SR8" },
      { id: "SR9-AFC-OTP-00001", station: "SR9" }, { id: "SR9-AFC-OTP-00002", station: "SR9" },
      { id: "SR11-AFC-OTP-00001", station: "SR11" }, { id: "SR11-AFC-OTP-00002", station: "SR11" },
      { id: "SR12-AFC-OTP-00001", station: "SR12" }, { id: "SR12-AFC-OTP-00002", station: "SR12" },
    ];
    const STAFFNAMES = [
      { label: "Nur Adibah binti Abdul Wahab", code: "900001" },
      { label: "Ali", code: "900002" },
      { label: "Gogo", code: "900003" },
    ];
    const SHIFTS = [{ label: "Morning", code: "Morning" }, { label: "Evening", code: "Evening" }];
    const CDM_LOCATIONS = [
      { label: "Rembus Depot 1", code: "Rembus-Depot-1" },
      { label: "Simpang Tiga 1", code: "Simpang-Tiga-1" },
      { label: "Simpang Tiga 2", code: "Simpang-Tiga-2" },
      { label: "Hikmah Exchange 1", code: "Hikmah-Exchange-1" },
      { label: "Hikmah Exchange 2", code: "Hikmah-Exchange-2" },
      { label: "Stadium 1", code: "Stadium-1" },
      { label: "Stadium 2", code: "Stadium-2" },
    ];

    /* === Helpers === */
    const pad2 = (n) => String(n).padStart(2, "0");
    const fmtDateYYYYMMDD = (date) => { const d = new Date(date || new Date()); return `${d.getFullYear()}${pad2(d.getMonth()+1)}${pad2(d.getDate())}`; };
    const sanitizeForFilename = (s) => (s||"").toString().trim().replace(/[\/\\\s]+/g,"-").replace(/[^A-Za-z0-9._-]/g,"");
    const humanSize = (b)=>{if(!Number.isFinite(b))return"-";const u=["B","KB","MB","GB","TB"];let i=0,v=b;while(v>=1024&&i<u.length-1){v/=1024;i++;}return `${v.toFixed(v>=10||i===0?0:1)} ${u[i]}`;};
    const isAllowedFile = (file) => file && (file.type.startsWith("image/") || file.type === "application/pdf");

    /* === DOM === */
    const btnPage1 = document.getElementById("btn-page1");
    const btnPage2 = document.getElementById("btn-page2");
    const btnPage3 = document.getElementById("btn-page3");
    const btnPage4 = document.getElementById("btn-page4");
    const panels = {
      page1: document.getElementById("panel-page1"),
      page2: document.getElementById("panel-page2"),
      page3: document.getElementById("panel-page3"),
      page4: document.getElementById("panel-page4"),
    };
    const backBtn = document.getElementById("backBtn");
    const nextBtn = document.getElementById("nextBtn");
    const submitBtn = document.getElementById("submitBtn");

    const staffSearch = document.getElementById("staffSearch");
    const staffNameSel = document.getElementById("staffName");
    const lineSel = document.getElementById("line");
    const stationSel = document.getElementById("station");
    const equipSel = document.getElementById("equip");
    const equipPreview = document.getElementById("equipPreview");
    const shiftSel = document.getElementById("shift");
    const cdmSel = document.getElementById("cdm");
    const dateInput = document.getElementById("theDate");
    const customInput = document.getElementById("shiftNo");

    // uploads
    const dropzone = document.getElementById("dropzone");
    const chooseFilesBtn = document.getElementById("chooseFilesBtn");
    theCaptureBtn = document.getElementById("captureBtn");
    const fileInput = document.getElementById("fileInput");
    const cameraInput = document.getElementById("cameraInput");
    const filesListEl = document.getElementById("filesList");
    const filesSummaryEl = document.getElementById("filesSummary");
    let uploadedFiles = []; // {file, id}

    // review / email
    const filenameOut = document.getElementById("filename");
    const copyBtn = document.getElementById("copyBtn");
    const resetBtn = document.getElementById("resetBtn");
    const emailTo = document.getElementById("emailTo");
    const emailCc = document.getElementById("emailCc");
    const emailSubject = document.getElementById("emailSubject");
    const emailMessage = document.getElementById("emailMessage");
    const sendEmailBtn = document.getElementById("sendEmailBtn");
    const downloadPdfBtn = document.getElementById("downloadPdfBtn");
    const toastEl = document.getElementById("toast");

    /* === Populate === */
    function initLines(){ lineSel.innerHTML = `<option value="" disabled selected>— Choose line —</option>` + LINES.map(s=>`<option value="${s.code}">${s.label} (${s.code})</option>`).join(""); }
    function populateStations(lineCode){ const f=STATIONS.filter(s=>s.line===lineCode); stationSel.innerHTML = `<option value="" disabled selected>— Choose station —</option>` + f.map(s=>`<option value="${s.code}">${s.label} (${s.code})</option>`).join(""); }
    function populateEquipments(stationCode){ const f=EQUIPMENTS.filter(e=>e.station===stationCode); equipSel.innerHTML = `<option value="" disabled selected>— Choose equipment —</option>` + f.map(e=>`<option value="${e.id}">${e.id}</option>`).join(""); }
    function initShift(){ shiftSel.innerHTML = `<option value="" disabled selected>— Choose shift —</option>` + SHIFTS.map(s=>`<option value="${s.code}">${s.label}</option>`).join(""); }
    function initCDM(){ cdmSel.innerHTML = `<option value="" disabled selected>— Choose CDM —</option>` + CDM_LOCATIONS.map(s=>`<option value="${s.code}">${s.label}</option>`).join(""); }
    function populateStaff(filterText=""){ const f=filterText.trim().toLowerCase(); const opts=STAFFNAMES.filter(s=>!f||s.label.toLowerCase().includes(f)||(s.code&&s.code.toLowerCase().includes(f))); const cur=staffNameSel.value; staffNameSel.innerHTML = `<option value="" disabled ${cur?"":"selected"}>— Choose staff —</option>` + opts.map(s=>`<option value="${s.code}">${s.label} (${s.code})</option>`).join(""); if(cur&&opts.some(o=>o.code===cur)) staffNameSel.value=cur; }
    function initDate(){ const t=new Date(); dateInput.value = `${t.getFullYear()}-${pad2(t.getMonth()+1)}-${pad2(t.getDate())}`; }

    /* === Filename === */
    function buildFilename(){
      const parts = [
        sanitizeForFilename(lineSel.value || "LINE"),
        sanitizeForFilename(stationSel.value || "STN"),
        sanitizeForFilename(equipSel.value || "EQUIP"),
        sanitizeForFilename(shiftSel.value || "Shift"),
        sanitizeForFilename(cdmSel.value || "CDM"),
        fmtDateYYYYMMDD(dateInput.value || new Date())
      ];
      const custom = sanitizeForFilename(customInput.value).replace(/^-+/, "");
      if (custom) parts.push(custom);
      return parts.join("_") + ".pdf";
    }
    function refreshPreview(){ equipPreview.textContent = equipSel.value || "—"; filenameOut.textContent = buildFilename(); syncEmailButton(); }

    /* === Uploads UI === */
    function renderFilesList(){
      filesListEl.innerHTML=""; let total=0;
      uploadedFiles.forEach(({file,id})=>{
        total += file.size;
        const row=document.createElement("div"); row.className="file-row";
        const typeLabel = file.type.startsWith("image/") ? "Image" : "PDF";
        row.innerHTML=`
          <div>
            <div>${file.name}</div>
            <div class="file-meta">${typeLabel} • ${humanSize(file.size)}</div>
          </div>
          <div class="file-actions">
            <button class="btn btn-rose" data-remove="${id}" type="button">Remove</button>
          </div>`;
        filesListEl.appendChild(row);
      });
      filesSummaryEl.textContent = uploadedFiles.length ? `${uploadedFiles.length} file(s), ${humanSize(total)} total` : "No files added yet.";
      filesListEl.querySelectorAll("[data-remove]").forEach(btn=>{
        btn.addEventListener("click",()=>{ const rid=btn.getAttribute("data-remove"); uploadedFiles = uploadedFiles.filter(x=>x.id!==rid); renderFilesList(); updateSidebarState(); syncFooterButtons(); });
      });
    }
    function addFiles(fileList){
      const arr = Array.from(fileList||[]); const rejected=[];
      arr.forEach(f=>{ if(!isAllowedFile(f)){ rejected.push(f.name); return; }
        const id = `${f.name}-${f.size}-${f.lastModified}-${Math.random().toString(36).slice(2,7)}`;
        uploadedFiles.push({file:f,id});
      });
      if (rejected.length) alert("Some files were skipped (only PDF or images are allowed):\n• " + rejected.join("\n• "));
      renderFilesList(); updateSidebarState(); syncFooterButtons();
    }
    ["dragenter","dragover","dragleave","drop"].forEach(evt=>{
      dropzone.addEventListener(evt,e=>{ e.preventDefault(); e.stopPropagation(); },false);
    });
    ["dragenter","dragover"].forEach(evt=> dropzone.addEventListener(evt,()=>dropzone.classList.add("dragover")));
    ["dragleave","drop"].forEach(evt=> dropzone.addEventListener(evt,()=>dropzone.classList.remove("dragover")));
    dropzone.addEventListener("drop", e=> addFiles(e.dataTransfer.files));
    dropzone.addEventListener("click", ()=> fileInput.click());
    chooseFilesBtn.addEventListener("click", ()=> fileInput.click());
    captureBtn.addEventListener("click", ()=> cameraInput.click());
    fileInput.addEventListener("change", e=>{ addFiles(e.target.files); fileInput.value=""; });
    cameraInput.addEventListener("change", e=>{ addFiles(e.target.files); cameraInput.value=""; });

    /* === Validation & Nav state === */
    function page1Valid(){ return staffNameSel.value && lineSel.value && stationSel.value && equipSel.value && shiftSel.value && cdmSel.value && dateInput.value; }
    function page2Valid(){ return !!customInput.value.trim(); }
    function page3Valid(){ return uploadedFiles.length > 0; }

    function updateSidebarState(){
      btnPage1.classList.toggle("done", page1Valid());
      btnPage2.classList.toggle("done", page2Valid());
      btnPage3.classList.toggle("done", page3Valid());
      btnPage2.disabled = !page1Valid();
      btnPage3.disabled = !(page1Valid() && page2Valid());
      btnPage4.disabled = !(page1Valid() && page2Valid() && page3Valid());
    }
    function setActive(sectionKey){
      Object.values(panels).forEach(p=>p.classList.remove("active"));
      panels[sectionKey].classList.add("active");
      [btnPage1,btnPage2,btnPage3,btnPage4].forEach(b=>b.classList.remove("active"));
      ({page1:btnPage1,page2:btnPage2,page3:btnPage3,page4:btnPage4})[sectionKey].classList.add("active");
      syncFooterButtons();
    }
    function currentSection(){
      if (panels.page1.classList.contains("active")) return "page1";
      if (panels.page2.classList.contains("active")) return "page2";
      if (panels.page3.classList.contains("active")) return "page3";
      return "page4";
    }
    function syncFooterButtons(){
      const sec=currentSection();
      backBtn.disabled = (sec==="page1");
      submitBtn.style.display = (sec==="page4") ? "inline-block" : "none";
      nextBtn.style.display = (sec==="page4") ? "none" : "inline-block";
      if (sec==="page1") { nextBtn.textContent="Next →"; nextBtn.disabled=!page1Valid(); }
      else if (sec==="page2") { nextBtn.textContent="Next →"; nextBtn.disabled=!page2Valid(); }
      else if (sec==="page3") { nextBtn.textContent="Next →"; nextBtn.disabled=!page3Valid(); }
    }

    /* === Actions === */
    function resetAll(){
      staffSearch.value=""; populateStaff(""); staffNameSel.value="";
      lineSel.value=""; stationSel.innerHTML=`<option value="" disabled selected>— Choose station —</option>`;
      equipSel.innerHTML=`<option value="" disabled selected>— Choose equipment —</option>`;
      shiftSel.value=""; cdmSel.value=""; initDate(); customInput.value="";
      uploadedFiles=[]; renderFilesList(); refreshPreview(); updateSidebarState(); setActive("page1");
      emailTo.value=""; emailCc.value=""; emailSubject.value=""; emailMessage.value="";
      syncEmailButton();
    }
    function copyTextToClipboard(text, btn, label="Copy"){
      navigator.clipboard.writeText(text).then(()=>{ btn.textContent="Copied!"; setTimeout(()=>btn.textContent=label,1200); })
        .catch(()=> alert("Copy failed. Please copy manually."));
    }

    /* === Toast === */
    let toastTimer = null;
    function toast(msg, ms=1800){
      toastEl.textContent = msg;
      toastEl.classList.add('show');
      clearTimeout(toastTimer);
      toastTimer = setTimeout(()=> toastEl.classList.remove('show'), ms);
    }

    /* === PDF helpers reused === */
    async function generatePDFReportBlob(){
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF({ unit:"mm", format:"a4", compress:true });
      const margin=12, lineH=6; let y=margin;

      // Header
      doc.setFont("helvetica","bold"); doc.setFontSize(16);
      doc.text("Cash Collection Report", margin, y); y+=lineH+2;
      doc.setFont("helvetica","normal"); doc.setFontSize(11);

      const kv = [
        ["Staff", (staffNameSel.selectedOptions[0]?.text || "-")],
        ["Date", fmtDateYYYYMMDD(dateInput.value)],
        ["Line", lineSel.value || "-"],
        ["Station", stationSel.value || "-"],
        ["Equipment", equipSel.value || "-"],
        ["Shift", shiftSel.value || "-"],
        ["CDM", cdmSel.value || "-"],
        ["Shift No", customInput.value || "-"]
      ];
      kv.forEach(([k,v])=>{ doc.text(`${k}: ${v}`, margin, y); y+=lineH; });

      // --- Sale Details (Summary) ---
      y+=2; doc.setFont("helvetica","bold"); doc.text("Sale Details", margin, y); y+=lineH;
      doc.setFont("helvetica","normal");
      doc.text(`Cash from Denominations: RM ${fmt(parseFloat(document.getElementById("cashDenomTotalAmt").value||"0"))}`, margin, y); y+=lineH;

      const pmIds = ["tot_cash","tot_cc","tot_dc","tot_spay","tot_ewallet","tot_duitnow"];
      const pmLabels = ["Cash","Credit Card","Debit Card","Spay","Other E-Wallet","DuitNow"];
      pmIds.forEach((id,i)=>{ const v=document.getElementById(id).value||"0.00"; doc.text(`${pmLabels[i]}: RM ${v}`, margin, y); y+=lineH; });

      doc.text(`Grand Total (Sales − Refunds): RM ${document.getElementById("grandTotal").value || "0.00"}`, margin, y); y+=lineH;

      // Product totals
      y+=2; doc.setFont("helvetica","bold"); doc.text("Per-Product Totals", margin, y); y+=lineH;
      doc.setFont("helvetica","normal");
      productTables.forEach(pt=>{
        doc.text(`${pt.name}: RM ${pt.rowTotal.value || "0.00"}`, margin, y); y+=lineH;
        if (y>280){ doc.addPage(); y=margin; }
      });

      // Reconciliation
      const reconText = document.getElementById("kpiRecon").textContent;
      if (y>280){ doc.addPage(); y=margin; }
      doc.text(`Reconciliation: ${reconText}`, margin, y); y+=lineH+2;

      // Attachments summary
      const imgFiles = uploadedFiles.filter(f=>f.file.type.startsWith("image/"));
      const pdfFiles = uploadedFiles.filter(f=>f.file.type==="application/pdf");
      doc.setFont("helvetica","bold"); doc.text("Attachments", margin, y); y+=lineH;
      doc.setFont("helvetica","normal");
      doc.text(`Images: ${imgFiles.length}`, margin, y); y+=lineH;
      if (pdfFiles.length){
        doc.text("PDFs:", margin, y); y+=lineH;
        pdfFiles.forEach(({file})=>{
          const name = file.name.length>60 ? file.name.slice(0,57)+"..." : file.name;
          doc.text(`• ${name} (${humanSize(file.size)})`, margin+4, y); y+=lineH;
          if (y>280){ doc.addPage(); y=margin; }
        });
      }

      // Embed images, one per page
      for (let i=0;i<imgFiles.length;i++){
        const dataUrl = await downscaleImageToDataURL(imgFiles[i].file, 1800, "image/jpeg", 0.85);
        const img = await loadImage(dataUrl);
        const pageW = doc.internal.pageSize.getWidth(), pageH = doc.internal.pageSize.getHeight();
        const maxW = pageW - margin*2, maxH = pageH - margin*2;
        const scale = Math.min(maxW/img.width, maxH/img.height);
        const w = Math.max(10, img.width*scale), h = Math.max(10, img.height*scale);
        const x = (pageW - w)/2, yy = (pageH - h)/2;
        doc.addPage();
        doc.setFont("helvetica","bold"); doc.setFontSize(12);
        doc.text(`Image ${i+1}/${imgFiles.length}: ${imgFiles[i].file.name}`, margin, margin-3+8);
        doc.addImage(dataUrl, "JPEG", x, yy, w, h);
      }

      const blob = doc.output('blob');
      return { blob, dataUri: doc.output('datauristring') };
    }

    async function generateAndDownloadPDF(){
      const { blob } = await generatePDFReportBlob();
      const a = document.createElement('a');
      a.href = URL.createObjectURL(blob);
      a.download = buildFilename();
      document.body.appendChild(a);
      a.click();
      URL.revokeObjectURL(a.href);
      a.remove();
    }

    function loadImage(dataUrl){ return new Promise((res,rej)=>{ const img=new Image(); img.onload=()=>res(img); img.onerror=rej; img.src=dataUrl; }); }
    async function downscaleImageToDataURL(file,maxDim=1800,outType="image/jpeg",quality=0.85){
      const dataUrl = await new Promise((res,rej)=>{ const fr=new FileReader(); fr.onload=()=>res(fr.result); fr.onerror=rej; fr.readAsDataURL(file); });
      const img = await loadImage(dataUrl);
      const scale = Math.min(1, maxDim/Math.max(img.width,img.height));
      const w=Math.max(1,Math.round(img.width*scale)), h=Math.max(1,Math.round(img.height*scale));
      const canvas=document.createElement("canvas"); canvas.width=w; canvas.height=h;
      const ctx=canvas.getContext("2d"); ctx.drawImage(img,0,0,w,h);
      return canvas.toDataURL(outType,quality);
    }

    /* === PAGE 2: Denominations & Sales Matrix & Product Subtables === */
    const cashDenomTable = document.getElementById("cashDenomTable");
    const cashDenomTotalQty = document.getElementById("cashDenomTotalQty");
    const cashDenomTotalAmt = document.getElementById("cashDenomTotalAmt");
    const kpiCashDenom = document.getElementById("kpiCashDenom");
    const kpiCashMatrix = document.getElementById("kpiCashMatrix");
    const kpiRecon = document.getElementById("kpiRecon");
    const salesMatrix = document.getElementById("salesMatrix");
    const productSubtablesHost = document.getElementById("productSubtables");

    const pmKeys = ["cash","cc","dc","spay","ewallet","duitnow"];
    const pmLabelsMap = { cash:"Cash", cc:"Credit Card", dc:"Debit Card", spay:"Spay", ewallet:"Other E-Wallet", duitnow:"DuitNow" };
    const pmIds  = {
      cash: "tot_cash", cc: "tot_cc", dc: "tot_dc",
      spay: "tot_spay", ewallet: "tot_ewallet", duitnow: "tot_duitnow"
    };
    const rowKeys = ["tng_card_sale","tng_card_sale_reload","tng_reload_only","refund"];

    const PRODUCTS = [
      { key:"sjt", name:"Single Journey Ticket" },
      { key:"rjt", name:"Return Journey Ticket" },
      { key:"pass", name:"Transit Pass" }
    ];
    let productTables = []; // { key, elem, inputs:[HTMLInputElement...], rowTotal }

    function toRM(n){ return (Math.round((n + Number.EPSILON) * 100)/100); }
    function fmt(n){ return toRM(n).toFixed(2); }

    /* Build product subtables dynamically */
    function buildProductSubtables(){
      productSubtablesHost.innerHTML = "";
      productTables = [];

      PRODUCTS.forEach(prod=>{
        const wrap = document.createElement("div");
        wrap.className = "subsection";
        wrap.innerHTML = `
          <h4>${prod.name}</h4>
          <div class="table-wrap">
            <table class="table" aria-label="${prod.name}">
              <thead>
                <tr>
                  <th>Payment Method</th>
                  <th class="num">Amount (RM)</th>
                </tr>
              </thead>
              <tbody>
                ${pmKeys.map(k=>`
                  <tr data-pm="${k}">
                    <td>${pmLabelsMap[k]}</td>
                    <td class="num"><input type="number" min="0" step="0.01" value="0" /></td>
                  </tr>
                `).join("")}
              </tbody>
              <tfoot>
                <tr>
                  <td style="text-align:right">Product Total</td>
                  <td class="num"><input type="number" data-total="row" readonly /></td>
                </tr>
              </tfoot>
            </table>
          </div>
        `;
        productSubtablesHost.appendChild(wrap);

        const table = wrap.querySelector("table");
        const inputs = Array.from(table.querySelectorAll('tbody input[type="number"]'));
        const rowTotal = table.querySelector('tfoot input[data-total="row"]');

        productTables.push({ key: prod.key, name: prod.name, elem: table, inputs, rowTotal });
      });

      // Wire listeners
      productTables.forEach(pt=>{
        pt.inputs.forEach(inp=> inp.addEventListener("input", ()=>{ recalcProducts(); recalcMatrix(); }));
      });
    }

    /* Denomination calc */
    function recalcDenoms(){
      let totalQty=0, totalAmt=0;
      cashDenomTable.querySelectorAll("tbody tr").forEach(tr=>{
        const denom = parseFloat(tr.getAttribute("data-denom"));
        const qtyInput = tr.querySelectorAll("input")[0];
        const amtInput = tr.querySelectorAll("input")[1];
        const qty = Math.max(0, parseInt(qtyInput.value||"0",10));
        const amt = toRM(qty * denom);
        amtInput.value = fmt(amt);
        totalQty += qty; totalAmt += amt;
      });
      cashDenomTotalQty.value = totalQty;
      cashDenomTotalAmt.value = fmt(totalAmt);
      kpiCashDenom.textContent = fmt(totalAmt);
      reconcile();
    }
    cashDenomTable.querySelectorAll('tbody tr input[type="number"]').forEach(inp=>{
      inp.addEventListener("input", recalcDenoms);
    });

    /* Read matrix row inputs */
    function getRowInputs(tr){
      const inputs = Array.from(tr.querySelectorAll('td input[type="number"]'));
      return { edits: inputs.slice(0,6), rowTotal: inputs[6] };
    }

    /* Product subtables calc — returns per-payment totals from products */
    function recalcProducts(){
      let productColTotals = {cash:0, cc:0, dc:0, spay:0, ewallet:0, duitnow:0};

      productTables.forEach(pt=>{
        let sum = 0;
        const rows = pt.elem.querySelectorAll('tbody tr');
        rows.forEach((tr,i)=>{
          const inp = tr.querySelector('input[type="number"]');
          const v = Math.max(0, parseFloat(inp.value||"0"));
          sum += v;
          productColTotals[pmKeys[i]] += v;
        });
        pt.rowTotal.value = fmt(sum);
      });

      return productColTotals;
    }

    /* Sales matrix calc (TNG + Refund + Product subtables) */
    function recalcMatrix(){
      const productColTotals = recalcProducts();

      let colTotals = {cash:0, cc:0, dc:0, spay:0, ewallet:0, duitnow:0};
      let grand = 0;

      salesMatrix.querySelectorAll("tbody tr").forEach((tr)=>{
        const rowKey = tr.getAttribute("data-row");
        const {edits, rowTotal} = getRowInputs(tr);

        let rowSum = 0;
        edits.forEach((inp, i)=>{
          const v = Math.max(0, parseFloat(inp.value||"0"));
          rowSum += v;
          colTotals[pmKeys[i]] += v;
        });

        rowTotal.value = fmt(rowSum);
        if (rowKey === "refund") grand -= rowSum; else grand += rowSum;
      });

      // Add product subtables into grand & columns
      pmKeys.forEach(k=> colTotals[k] += productColTotals[k]);
      const productGrand = Object.values(productColTotals).reduce((a,b)=>a+b,0);
      grand += productGrand;

      // Subtract refund per method from col totals
      const refundTr = salesMatrix.querySelector('tbody tr[data-row="refund"]');
      const refundEdits = getRowInputs(refundTr).edits;
      refundEdits.forEach((inp, i)=>{
        const v = Math.max(0, parseFloat(inp.value||"0"));
        colTotals[pmKeys[i]] -= v;
      });

      // Write totals
      Object.entries(pmIds).forEach(([k,id])=>{
        const el = document.getElementById(id);
        el.value = fmt(colTotals[k]);
      });
      document.getElementById("grandTotal").value = fmt(grand);

      // KPI cash (matrix) after refunds
      kpiCashMatrix.textContent = fmt(colTotals.cash);
      reconcile();
    }

    // Wire matrix listeners
    salesMatrix.querySelectorAll('tbody tr input[type="number"]').forEach(inp=>{
      inp.addEventListener("input", recalcMatrix);
    });

    /* Reconciliation */
    function reconcile(){
      const denom = parseFloat(cashDenomTotalAmt.value||"0");
      const matrixCash = parseFloat(document.getElementById("tot_cash").value||"0");
      const diff = toRM(matrixCash - denom);
      if (Math.abs(diff) < 0.01){
        kpiRecon.textContent = "Balanced";
        kpiRecon.className = "status ok";
      } else {
        kpiRecon.textContent = `Diff RM ${fmt(diff)}`;
        kpiRecon.className = "status bad";
      }
    }

    /* === Generate & Email === */
    async function sendEmailWithAttachment(){
      const to = emailTo.value.trim();
      if (!to) { alert("Please fill the recipient email."); return; }
      const cc = emailCc.value.trim();
      const subj = (emailSubject.value || "Cash Collection Report").trim();
      const msg  = (emailMessage.value || "Please find the attached Cash Collection Report.").trim();

      toast("Building PDF…");
      const { blob, dataUri } = await generatePDFReportBlob();
      const filename = buildFilename();

      // Try EmailJS if configured
      const emailJsConfigured = EMAILJS_PUBLIC_KEY && EMAILJS_SERVICE_ID && EMAILJS_TEMPLATE_ID && (typeof emailjs !== "undefined");
      if (emailJsConfigured) {
        try {
          // Extract base64 (strip data URI prefix)
          const base64 = dataUri.split(",")[1];

          sendEmailBtn.disabled = true;
          sendEmailBtn.textContent = "Sending…";

          const params = {
            to_email: to,
            cc_email: cc || "",
            email_subject: subj,
            email_message: msg,
            attachment: base64,                // map in template as attachment
            attachment_filename: filename      // and filename
          };

          await emailjs.send(EMAILJS_SERVICE_ID, EMAILJS_TEMPLATE_ID, params);

          // Also download locally for record
          const a = document.createElement('a');
          a.href = URL.createObjectURL(blob);
          a.download = filename;
          document.body.appendChild(a); a.click(); URL.revokeObjectURL(a.href); a.remove();

          toast("Email sent and PDF downloaded.");
          sendEmailBtn.textContent = "Sent ✓";
        } catch (err) {
          console.error(err);
          alert("EmailJS failed. We’ll open your email client without the attachment, and download the PDF so you can attach it manually.");
          await fallbackMailtoAndDownload(blob, filename, to, cc, subj, msg);
          sendEmailBtn.textContent = "Send Email (attach PDF)";
        } finally {
          sendEmailBtn.disabled = false;
        }
        return;
      }

      // Fallback: mailto + download
      await fallbackMailtoAndDownload(blob, filename, to, cc, subj, msg);
    }

    async function fallbackMailtoAndDownload(blob, filename, to, cc, subj, msg){
      // Download PDF
      const a = document.createElement('a');
      a.href = URL.createObjectURL(blob);
      a.download = filename;
      document.body.appendChild(a); a.click(); URL.revokeObjectURL(a.href); a.remove();

      // Open default mail client (cannot auto-attach)
      const mailto = `mailto:${encodeURIComponent(to)}?subject=${encodeURIComponent(subj)}${cc?`&cc=${encodeURIComponent(cc)}`:""}&body=${encodeURIComponent(msg + "\n\n(Attached: " + filename + ")")}`;
      window.location.href = mailto;
      toast("PDF downloaded. Email window opened.");
    }

    function syncEmailButton(){
      sendEmailBtn.disabled = !(emailTo.value.trim() && page1Valid() && page2Valid() && page3Valid());
    }

    /* === Events: nav === */
    document.getElementById("btn-page1").addEventListener("click", ()=> setActive("page1"));
    document.getElementById("btn-page2").addEventListener("click", ()=> { if(!btnPage2.disabled) setActive("page2"); });
    document.getElementById("btn-page3").addEventListener("click", ()=> { if(!btnPage3.disabled) setActive("page3"); });
    document.getElementById("btn-page4").addEventListener("click", ()=> { if(!btnPage4.disabled) setActive("page4"); });

    backBtn.addEventListener("click", ()=>{
      const sec=currentSection();
      if (sec==="page2") setActive("page1");
      else if (sec==="page3") setActive("page2");
      else if (sec==="page4") setActive("page3");
    });
    nextBtn.addEventListener("click", ()=>{
      const sec=currentSection();
      if (sec==="page1" && page1Valid()) setActive("page2");
      else if (sec==="page2" && page2Valid()) setActive("page3");
      else if (sec==="page3" && page3Valid()) setActive("page4");
    });

    submitBtn.addEventListener("click", async ()=>{
      try { await generateAndDownloadPDF(); toast("PDF generated."); }
      catch(err){ console.error(err); alert("Could not generate the PDF. Try with fewer/lower-resolution images."); }
    });

    downloadPdfBtn.addEventListener("click", async ()=>{
      try { await generateAndDownloadPDF(); toast("PDF downloaded."); }
      catch(err){ console.error(err); alert("Could not generate the PDF. Try with fewer/lower-resolution images."); }
    });

    sendEmailBtn.addEventListener("click", sendEmailWithAttachment);

    /* === Events: form === */
    staffSearch.addEventListener("input", e=>{ populateStaff(e.target.value); updateSidebarState(); syncFooterButtons(); });
    staffNameSel.addEventListener("change", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    lineSel.addEventListener("change", ()=>{ populateStations(lineSel.value); stationSel.value=""; equipSel.innerHTML=`<option value="" disabled selected>— Choose equipment —</option>`; updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    stationSel.addEventListener("change", ()=>{ populateEquipments(stationSel.value); equipSel.value=""; updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    equipSel.addEventListener("change", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    shiftSel.addEventListener("change", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    cdmSel.addEventListener("change", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    dateInput.addEventListener("change", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    customInput.addEventListener("input", ()=>{ updateSidebarState(); syncFooterButtons(); refreshPreview(); });
    emailTo.addEventListener("input", syncEmailButton);

    copyBtn.addEventListener("click", ()=> copyTextToClipboard(buildFilename(), copyBtn, "Copy Name"));
    resetBtn.addEventListener("click", resetAll);

    /* === Boot === */
    (function boot(){
      initLines(); initShift(); initCDM(); populateStaff(""); initDate();
      refreshPreview(); updateSidebarState(); setActive("page1"); renderFilesList();
      buildProductSubtables(); recalcDenoms(); recalcProducts(); recalcMatrix();
      emailSubject.value = "Cash Collection Report";
      emailMessage.value = "Please find the attached Cash Collection Report.";
    })();
  </script>
</body>
</html>
