# COAL-project
Assembly visualizer
// ----------------------------------------------------
// 1. Textarea Line Numbers Sync
// ----------------------------------------------------
const editor = document.getElementById('code-editor');
const lineNumbers = document.getElementById('line-numbers');

function updateLineNumbers() {
    const lines = editor.value.split('\n').length;
    let lineNumbersHtml = '';
    for (let i = 1; i <= Math.max(1, lines); i++) {
        lineNumbersHtml += `<div id="line-${i}" class="transition-colors">${i}</div>`;
    }
    lineNumbers.innerHTML = lineNumbersHtml;
}

editor.addEventListener('input', updateLineNumbers);

editor.addEventListener('scroll', () => {
    lineNumbers.scrollTop = editor.scrollTop;
});

updateLineNumbers();

// ----------------------------------------------------
// 2. Memory Table Initialization
// ----------------------------------------------------
const memoryTable = document.getElementById('memory-table');

function initMemory() {
    memoryTable.innerHTML = '';
    for (let i = 0; i < 16; i++) {
        const addressHex = i.toString(16).padStart(4, '0').toUpperCase();
        addMemoryRow(addressHex, '00');
    }
}

function addMemoryRow(addressHex, valueHex) {
    const row = document.createElement('tr');
    row.className = 'border-b border-white/5 hover:bg-white/5 transition-colors duration-200';
    row.id = `mem-${addressHex}`;
    row.innerHTML = `
        <td class="px-5 py-2.5 text-slate-400 border-r border-white/5 font-mono text-xs tracking-widest">0x${addressHex}</td>
        <td class="px-5 py-2.5 text-cyan-400 mem-value font-mono text-xs tracking-widest">${valueHex}</td>
    `;
    memoryTable.appendChild(row);
    return row;
}

initMemory();

// ----------------------------------------------------
// 3. Main UI Update Function (The Core Requirement)
// ----------------------------------------------------
/**
 * Updates the UI with the provided CPU State JSON object.
 */
window.updateUI = function(cpuState) {
    const allLines = lineNumbers.children;
    for (let i = 0; i < allLines.length; i++) {
        allLines[i].className = 'text-slate-600 transition-colors';
    }
    
    if (cpuState.current_line && cpuState.current_line > 0) {
        const activeLineEl = document.getElementById(`line-${cpuState.current_line}`);
        if (activeLineEl) {
            activeLineEl.className = 'text-cyan-400 font-bold bg-cyan-900/30 border-l-2 border-cyan-400 -ml-[2px] pl-[6px] transition-colors';
        }
    }
    if (cpuState.registers) {
        const regIds = ['AX', 'BX', 'CX', 'DX', 'IP'];
        regIds.forEach(reg => {
            if (cpuState.registers[reg] !== undefined) {
                const el = document.getElementById(`reg-${reg.toLowerCase()}`);
                if (el) el.textContent = cpuState.registers[reg];
            }
        });
    }
    if (cpuState.flags) {
        const flagIds = ['ZF', 'CF', 'SF', 'OF'];
        flagIds.forEach(flag => {
            if (cpuState.flags[flag] !== undefined) {
                const el = document.getElementById(`flag-${flag.toLowerCase()}`);
                if (!el) return;
                
                const val = cpuState.flags[flag];
                if (val === 1) {
                    // Turn on (Neon Green glowing LED)
                    el.className = 'w-12 h-3 rounded-full bg-emerald-400 shadow-[0_0_12px_rgba(52,211,153,0.8)] border border-emerald-300/50 transition-all duration-300';
                } else {
                    // Turn off (Dark muted glass pill)
                    el.className = 'w-12 h-3 rounded-full bg-slate-800 border border-white/5 shadow-inner transition-all duration-300';
                }
            }
        });
    }

    // -- 3d. Update Memory --
    if (cpuState.memory) {
        for (const [address, value] of Object.entries(cpuState.memory)) {
            const addrUpper = address.toUpperCase();
            const rowId = `mem-${addrUpper}`;
            let row = document.getElementById(rowId);
            
            if (row) {
                const valCell = row.querySelector('.mem-value');
                if (valCell) valCell.textContent = value;
                
                // Cyan pulse effect on change
                row.classList.add('bg-cyan-900/40', 'shadow-[inset_0_0_15px_rgba(6,182,212,0.3)]');
                setTimeout(() => row.classList.remove('bg-cyan-900/40', 'shadow-[inset_0_0_15px_rgba(6,182,212,0.3)]'), 400);
            } else {
                const newRow = addMemoryRow(addrUpper, value);
                newRow.classList.add('bg-cyan-900/40', 'shadow-[inset_0_0_15px_rgba(6,182,212,0.3)]');
                setTimeout(() => newRow.classList.remove('bg-cyan-900/40', 'shadow-[inset_0_0_15px_rgba(6,182,212,0.3)]'), 400);
            }
        }
    }
};

// ----------------------------------------------------
// 4. Backend Integration
// ----------------------------------------------------
const API_URL = "http://127.0.0.1:5000/execute"; 
const RESET_URL = "http://127.0.0.1:5000/reset";

async function sendToBackend(actionType) {
    const code = document.getElementById('code-editor').value;
    const btn = document.getElementById(`btn-${actionType}`);
    const originalText = btn.textContent;
    btn.textContent = "WAIT...";
    
    try {
        const response = await fetch(API_URL, {
            method: "POST",
            headers: {
                "Content-Type": "application/json"
            },
            body: JSON.stringify({ 
                code: code,
                action: actionType 
            })
        });

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const backendData = await response.json();
        updateUI(backendData);

    } catch (error) {
        console.error("Backend Error:", error);
        alert("Failed to connect to backend. Is Ayesha's server running?");
    } finally {
        btn.textContent = originalText;
    }
}

document.getElementById('btn-step').addEventListener('click', () => {
    sendToBackend("step");
});

document.getElementById('btn-run').addEventListener('click', () => {
    sendToBackend("run");
});

document.getElementById('btn-reset').addEventListener('click', () => {
    // 1. Reset the frontend visuals
    updateUI({
        "current_line": -1, 
        "registers": {"AX": "0000", "BX": "0000", "CX": "0000", "DX": "0000", "IP": "0000"},
        "flags": {"ZF": 0, "CF": 0, "SF": 0, "OF": 0},
        "memory": {} 
    });
    initMemory();
    fetch(RESET_URL, { method: "POST" })
        .catch(e => console.error("Failed to reset backend:", e));
});
