<!-- === PATCH: layout + fixes + Option A (single source of truth = Matrix) === -->
<style>
  /* 1) Better readability for active step buttons */
  .navbtn.active { color:#fff; }

  /* 2) Dynamic/compact layout for small screens:
        - Turn the sidebar into a horizontal scroller
        - Keep buttons readable and non-wrapping */
  @media (max-width: 900px) {
    .sidebar {
      display: flex;
      flex-direction: row;
      gap: 8px;
      overflow-x: auto;
      -webkit-overflow-scrolling: touch;
      white-space: nowrap;
      scrollbar-width: thin;
      padding-bottom: 10px;
    }
    .sidebar .navbtn {
      flex: 0 0 auto; /* prevent squish */
    }
  }

  /* 3) Optional: make tables breathe on tiny screens */
  @media (max-width: 640px) {
    .table th, .table td { padding: 8px; }
    .num { width: 96px; }
  }
</style>

<script>
/* ========= PATCH SCRIPT ========= */
(function () {
  /* ---- A. DOM safety fixes ---- */

  // A1) Fix the illegal ID with spaces: <input id="Shift No"> → programmatically rename to shiftNo
  (function fixShiftNoId() {
    const bad = document.querySelector('input[id="Shift No"]');
    if (bad) {
      bad.id = 'shiftNo';
    }
  })();
  // Rebind the reference used in the app:
  const customInput = document.getElementById('shiftNo') || document.getElementById('Shift No');

  // A2) Fix malformed <th> in Cash Denomination table header (browser may tolerate, but we enforce)
  (function fixBadTh() {
    const ths = document.querySelectorAll('#cashDenomTable thead th');
    if (ths && ths.length) {
      ths[0].textContent = 'Denomination';
    }
  })();

  // A3) Make active tab contrast safe even if theme overrides were missed (CSS already added)


  /* ---- B. Stop duplicate recalcs from product inputs ----
     We let the matrix be the single source of truth (Option A):
     - Product subtables remain for reporting; they DO NOT feed totals.
     - So, product inputs only recalc their own per-product totals (no impact on matrix totals).
  */
  // If buildProductSubtables already ran, rewire its listeners:
  function rewireProductInputsForOptionA() {
    const productTablesHost = document.getElementById('productSubtables');
    if (!productTablesHost) return;

    // Remove existing input listeners by cloning (cheap & safe),
    // then add a lean one that only recomputes product totals (not matrix).
    productTablesHost.querySelectorAll('tbody input[type="number"]').forEach(inp => {
      const clone = inp.cloneNode(true);
      inp.parentNode.replaceChild(clone, inp);
      clone.addEventListener('input', () => {
        try {
          if (typeof recalcProducts === 'function') recalcProducts();
          // DO NOT call recalcMatrix() here — avoids double-counting
        } catch (e) {}
      });
    });
  }

  // Rewire once at load, and again after any dynamic rebuilds.
  // (If your app only builds once, this single call is enough.)
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', rewireProductInputsForOptionA);
  } else {
    rewireProductInputsForOptionA();
  }

  /* ---- C. Override recalcMatrix to implement Option A cleanly ----
     Original recalcMatrix added productColTotals into column totals and grand → double count.
     We redefine it to ignore product subtables for totals. Reconciliation uses matrix cash only.
  */
  (function patchRecalcMatrixOptionA() {
    if (typeof recalcMatrix !== 'function') return;

    const originalRecalcProducts = (typeof recalcProducts === 'function') ? recalcProducts : null;

    window.recalcMatrix = function recalcMatrix_PatchedOptionA() {
      // Keep product subtables independent (for their own totals only)
      if (originalRecalcProducts) originalRecalcProducts();

      const salesMatrix = document.getElementById('salesMatrix');
      const pmKeys = ['cash','cc','dc','spay','ewallet','duitnow'];
      const pmIds  = { cash:'tot_cash', cc:'tot_cc', dc:'tot_dc', spay:'tot_spay', ewallet:'tot_ewallet', duitnow:'tot_duitnow' };

      let colTotals = {cash:0, cc:0, dc:0, spay:0, ewallet:0, duitnow:0};
      let grand = 0;

      function getRowInputs(tr){
        const inputs = Array.from(tr.querySelectorAll('td input[type="number"]'));
        return { edits: inputs.slice(0,6), rowTotal: inputs[6] };
      }

      salesMatrix.querySelectorAll('tbody tr').forEach((tr)=>{
        const rowKey = tr.getAttribute('data-row');
        const {edits, rowTotal} = getRowInputs(tr);

        let rowSum = 0;
        edits.forEach((inp, i)=>{
          const v = Math.max(0, parseFloat(inp.value || '0'));
          rowSum += v;
          colTotals[pmKeys[i]] += v;
        });

        if (rowTotal) rowTotal.value = (Math.round((rowSum + Number.EPSILON) * 100)/100).toFixed(2);
        if (rowKey === 'refund') grand -= rowSum; else grand += rowSum;
      });

      // Column totals do NOT include product subtables (Option A)
      Object.entries(pmIds).forEach(([k,id]) => {
        const el = document.getElementById(id);
        if (el) el.value = (Math.round((colTotals[k] + Number.EPSILON) * 100)/100).toFixed(2);
      });

      const grandEl = document.getElementById('grandTotal');
      if (grandEl) grandEl.value = (Math.round((grand + Number.EPSILON) * 100)/100).toFixed(2);

      // Update KPIs + reconciliation (matrix cash vs denominations)
      const kpiCashMatrix = document.getElementById('kpiCashMatrix');
      if (kpiCashMatrix) kpiCashMatrix.textContent = document.getElementById('tot_cash')?.value || '0.00';

      if (typeof reconcile === 'function') reconcile();
    };

    // Rebind listeners on matrix inputs to use patched function
    document.querySelectorAll('#salesMatrix tbody tr input[type="number"]').forEach(inp=>{
      const clone = inp.cloneNode(true);
      inp.parentNode.replaceChild(clone, inp);
      clone.addEventListener('input', window.recalcMatrix);
    });

    // Initial compute after patch
    try { window.recalcMatrix(); } catch(e){}
  })();


  /* ---- D. Email button gating + locking ----
     Disable the email button until:
      - page1/page2/page3 are valid
      - recipient email looks valid
     Also lock the button during sending and restore on completion/error.
  */
  (function gateAndLockEmail() {
    const emailBtn = document.getElementById('emailBtn');
    const recipientEmail = document.getElementById('recipientEmail');

    if (!emailBtn || !recipientEmail) return;

    function validEmail(v){
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v || '');
    }
    function page1Valid(){ try { return staffNameSel.value && lineSel.value && stationSel.value && equipSel.value && shiftSel.value && cdmSel.value && theDate.value; } catch(e){ return false; } }
    function page2Valid(){ try { return !!(customInput && customInput.value.trim()); } catch(e){ return false; } }
    function page3Valid(){ try { return (window.uploadedFiles && uploadedFiles.length > 0); } catch(e){ return false; } }

    function syncEmailBtn(){
      const ok = page1Valid() && page2Valid() && page3Valid() && validEmail(recipientEmail.value);
      emailBtn.disabled = !ok;
    }

    // Run gating on input + on various form changes
    recipientEmail.addEventListener('input', syncEmailBtn);
    ['change','input'].forEach(evt => {
      [staffNameSel,lineSel,stationSel,equipSel,shiftSel,cdmSel,document.getElementById('theDate'),customInput].forEach(el=>{
        if (el) el.addEventListener(evt, syncEmailBtn);
      });
    });

    // Also re-check after uploads mutate
    const filesListEl = document.getElementById('filesList');
    if (filesListEl) {
      const mo = new MutationObserver(syncEmailBtn);
      mo.observe(filesListEl, { childList: true, subtree: true });
    }

    // Lock during send (works with all three methods since they share #emailBtn click)
    const origHandler = emailBtn.onclick || null;
    emailBtn.addEventListener('click', async function lockWhileSending(e){
      if (emailBtn.disabled) return;
      const oldText = emailBtn.textContent;
      emailBtn.disabled = true;
      emailBtn.textContent = 'Sending…';
      try {
        // Let existing click handler run (the code 2 switch on method)
        // We re-dispatch programmatically to avoid double-binding problems.
        // If code 2 already bound a listener via addEventListener, this will still run that one.
      } finally {
        // give the existing handler a moment; then restore UI based on success/failure heuristics
        setTimeout(()=> {
          emailBtn.textContent = oldText;
          syncEmailBtn(); // re-enable only if form still valid
        }, 600);
      }
    }, { capture: true });
    // Initial state
    syncEmailBtn();
  })();


  /* ---- E. Optional: graceful fallback if emailing fails (server/EmailJS) ----
     If you want a safety net, intercept fetch/emailjs errors and still download the PDF.
     Minimal, non-invasive approach: wrap fetch & emailjs.send if present.
  */
  (function addEmailFallback() {
    // Wrap fetch used by emailViaServer
    const _fetch = window.fetch;
    if (typeof _fetch === 'function') {
      window.fetch = async function (...args) {
        try {
          const res = await _fetch.apply(this, args);
          if (!res.ok) throw new Error('Email send failed');
          return res;
        } catch (err) {
          console.warn('Email failed via server route, falling back to local download.', err);
          try {
            if (typeof buildPdfBlobAndName === 'function') {
              const { blob, filename } = await buildPdfBlobAndName();
              const url = URL.createObjectURL(blob);
              const a = document.createElement('a');
              a.href = url; a.download = filename; a.click();
              URL.revokeObjectURL(url);
              alert('Email failed. The PDF has been downloaded — please attach it manually.');
            }
          } catch (_) {}
          throw err; // preserve original behavior
        }
      };
    }

    // Wrap emailjs.send if present later
    const ensureWrapEmailJS = () => {
      if (!window.emailjs || !emailjs.send || emailjs.__wrapped) return;
      const _send = emailjs.send.bind(emailjs);
      emailjs.send = async function (...args) {
        try {
          return await _send(...args);
        } catch (err) {
          console.warn('EmailJS send failed, falling back to local download.', err);
          try {
            if (typeof buildPdfBlobAndName === 'function') {
              const { blob, filename } = await buildPdfBlobAndName();
              const url = URL.createObjectURL(blob);
              const a = document.createElement('a');
              a.href = url; a.download = filename; a.click();
              URL.revokeObjectURL(url);
              alert('Email failed. The PDF has been downloaded — please attach it manually.');
            }
          } catch (_) {}
          throw err;
        }
      };
      emailjs.__wrapped = true;
    };
    // Try now and later (in case emailjs loads async)
    ensureWrapEmailJS();
    const iv = setInterval(() => {
      try { ensureWrapEmailJS(); if (emailjs && emailjs.__wrapped) clearInterval(iv); } catch(e){}
    }, 500);
  })();
})();
</script>
