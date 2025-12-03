<html lang="th">
<head>
  <meta charset="UTF-8" />
  <title>ระบบแจ้งซ่อม โรงเรียนชุมชนมาบอำมฤต</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>

  <!-- ฟอนต์ Sarabun -->
  <link
    href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;500;600;700&display=swap"
    rel="stylesheet"
  />

  <style>
    body {
      font-family: "Sarabun", system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
    }
    .glass-panel {
      background: radial-gradient(circle at top left, rgba(34,211,238,0.25), transparent 60%),
                  radial-gradient(circle at bottom right, rgba(59,130,246,0.2), transparent 55%),
                  rgba(15,23,42,0.95);
      backdrop-filter: blur(16px);
      border: 1px solid rgba(148,163,184,0.35);
    }
    .hud-card {
      background: radial-gradient(circle at top, rgba(56,189,248,0.15), transparent 55%),
                  rgba(15,23,42,0.9);
      border-color: rgba(148,163,184,0.4);
    }
    .custom-scrollbar::-webkit-scrollbar {
      width: 6px;
    }
    .custom-scrollbar::-webkit-scrollbar-thumb {
      background: rgba(148,163,184,0.6);
      border-radius: 999px;
    }
    .custom-scrollbar::-webkit-scrollbar-track {
      background: transparent;
    }
  </style>
</head>
<body class="bg-slate-950 text-slate-100 min-h-screen">
  <div id="app" class="min-h-screen"></div>

  <script>
    // ---------------------------
    // ข้อมูลตัวอย่าง / State เริ่มต้น
    // ---------------------------
    const BUILDINGS = [
      { id: "B1", name: "อาคารเรียน 1" },
      { id: "B2", name: "อาคารเรียน 2" },
      { id: "B3", name: "อาคารวิทยาศาสตร์" },
      { id: "GYM", name: "โรงยิม / อาคารพละ" },
      { id: "LIB", name: "ห้องสมุด" },
    ];

    const TEAMS = [
      { id: 1, name: "ทีมช่างไฟฟ้า", status: "Available", currentIncidentId: null },
      { id: 2, name: "ทีมช่างประปา", status: "Available", currentIncidentId: null },
      { id: 3, name: "ทีมช่างทั่วไป", status: "Available", currentIncidentId: null },
    ];

    const USERS = [
      { username: "admin", password: "1234", role: "admin", name: "ผู้ดูแลระบบ" },
      { username: "tech1", password: "1234", role: "technician", name: "ทีมช่าง 1", teamId: 1 },
      { username: "tech2", password: "1234", role: "technician", name: "ทีมช่าง 2", teamId: 2 },
      { username: "teacher1", password: "1234", role: "teacher", name: "ครูตัวอย่าง (มัธยม)" },
    ];

    const PRIORITIES = ["Critical", "High", "Medium", "Low"];

    let state = {
      currentUser: null,
      incidents: [],
      teams: structuredClone(TEAMS),
      selectedBuildingId: null,
      filters: {
        status: "All",
        priority: "All",
        search: "",
      },
    };

    // โหลดจาก localStorage ถ้ามี
    const saved = localStorage.getItem("mab_ammarit_maintenance_state");
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        state = { ...state, ...parsed };
      } catch (e) {
        console.warn("โหลด state เก่าไม่สำเร็จ", e);
      }
    }

    function saveState() {
      const toSave = {
        currentUser: state.currentUser,
        incidents: state.incidents,
        teams: state.teams,
        selectedBuildingId: state.selectedBuildingId,
        filters: state.filters,
      };
      localStorage.setItem("mab_ammarit_maintenance_state", JSON.stringify(toSave));
    }

    // ---------------------------
    // Utility: เวลา
    // ---------------------------
    function formatRelativeTime(ts) {
      const now = Date.now();
      const diff = Math.floor((now - ts) / 1000);
      if (diff < 60) return "ไม่กี่วินาทีที่ผ่านมา";
      const mins = Math.floor(diff / 60);
      if (mins < 60) return `${mins} นาทีที่ผ่านมา`;
      const hrs = Math.floor(mins / 60);
      if (hrs < 24) return `${hrs} ชั่วโมงที่ผ่านมา`;
      const days = Math.floor(hrs / 24);
      return `${days} วันที่ผ่านมา`;
    }

    // ---------------------------
    // Render หลัก
    // ---------------------------
    const appEl = document.getElementById("app");

    function render() {
      if (!state.currentUser) {
        renderLogin();
      } else {
        renderDashboard();
      }
      saveState();
    }

    // ---------------------------
    // Login Screen
    // ---------------------------
    function renderLogin() {
      appEl.innerHTML = `
        <div class="min-h-screen flex items-center justify-center p-4 relative overflow-hidden">
          <div class="absolute inset-0 -z-10">
            <div class="absolute -top-32 -left-24 w-80 h-80 bg-cyan-500/20 rounded-full blur-3xl"></div>
            <div class="absolute -bottom-40 -right-32 w-96 h-96 bg-blue-500/20 rounded-full blur-3xl"></div>
          </div>
          <div class="glass-panel max-w-md w-full rounded-2xl shadow-2xl border-t border-l border-white/20">
            <div class="border-b border-white/10 px-8 pt-8 pb-6 text-center">
              <div class="flex justify-center mb-4">
                <div class="w-16 h-16 rounded-xl bg-cyan-500/10 border border-cyan-400 flex items-center justify-center shadow-lg">
                  <svg xmlns="http://www.w3.org/2000/svg" class="w-10 h-10 text-cyan-400" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                    <path d="M3 12L12 3l9 9" />
                    <path d="M5 10v10h14V10" />
                    <path d="M9 21V12h6v9" />
                  </svg>
                </div>
              </div>
              <h1 class="text-xl font-extrabold text-white tracking-wide">ระบบแจ้งซ่อมอุปกรณ์</h1>
              <p class="text-xs text-cyan-200/70 mt-1">โรงเรียนชุมชนมาบอำมฤต</p>
              <p class="text-[10px] text-slate-400 uppercase tracking-[0.3em] mt-2">School Facility Maintenance</p>
            </div>
            <div class="p-6 pb-8">
              <form id="loginForm" class="space-y-3">
                <div>
                  <label class="text-xs text-slate-400 font-semibold block mb-1">ชื่อผู้ใช้ (username)</label>
                  <input
                    id="loginUsername"
                    type="text"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2.5 text-sm text-white focus:outline-none focus:border-cyan-400"
                    placeholder="เช่น admin, tech1, teacher1"
                    required
                  />
                </div>
                <div>
                  <label class="text-xs text-slate-400 font-semibold block mb-1">รหัสผ่าน (password)</label>
                  <input
                    id="loginPassword"
                    type="password"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2.5 text-sm text-white focus:outline-none focus:border-cyan-400"
                    placeholder="เช่น 1234"
                    required
                  />
                </div>
                <div>
                  <label class="text-xs text-slate-400 font-semibold block mb-1">บทบาท</label>
                  <select
                    id="loginRole"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2.5 text-sm text-white focus:outline-none focus:border-cyan-400"
                  >
                    <option value="teacher">ครู / บุคลากร</option>
                    <option value="technician">ทีมช่าง</option>
                    <option value="admin">ผู้ดูแลระบบ</option>
                  </select>
                </div>
                <div id="loginError" class="text-xs text-red-400 min-h-[1.5rem]"></div>
                <button
                  type="submit"
                  class="w-full bg-cyan-600 hover:bg-cyan-500 text-white font-semibold py-2.5 rounded-lg text-sm shadow-lg transition-colors"
                >
                  เข้าสู่ระบบ
                </button>
              </form>

              <div class="mt-5 pt-4 border-t border-white/10 text-[11px] text-slate-400 space-y-1">
                <p class="font-semibold text-slate-300">บัญชีตัวอย่าง</p>
                <p>Admin → <span class="text-cyan-300">admin / 1234</span></p>
                <p>Technician → <span class="text-cyan-300">tech1 / 1234</span></p>
                <p>Teacher → <span class="text-cyan-300">teacher1 / 1234</span></p>
              </div>
            </div>
          </div>
        </div>
      `;

      const form = document.getElementById("loginForm");
      form.addEventListener("submit", (e) => {
        e.preventDefault();
        const username = document.getElementById("loginUsername").value.trim();
        const password = document.getElementById("loginPassword").value.trim();
        const role = document.getElementById("loginRole").value;
        const errorEl = document.getElementById("loginError");

        const user = USERS.find(
          (u) => u.username === username && u.password === password && u.role === role
        );
        if (!user) {
          errorEl.textContent = "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง หรือบทบาทไม่ตรงกัน";
          return;
        }
        errorEl.textContent = "";
        state.currentUser = {
          username: user.username,
          role: user.role,
          name: user.name,
          teamId: user.teamId || null,
        };
        render();
      });
    }

    // ---------------------------
    // Dashboard Layout
    // ---------------------------
    function renderDashboard() {
      const u = state.currentUser;
      const role = u.role;

      appEl.innerHTML = `
        <div class="min-h-screen flex flex-col bg-slate-950">
          <!-- Header -->
          <header class="sticky top-0 z-20 glass-panel border-b border-slate-800/80">
            <div class="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between gap-4">
              <div class="flex items-center gap-3">
                <div class="relative w-10 h-10 rounded-xl bg-cyan-500/20 flex items-center justify-center border border-cyan-400/40 shadow-lg">
                  <svg xmlns="http://www.w3.org/2000/svg" class="w-6 h-6 text-cyan-300" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                    <path d="M3 12L12 3l9 9" />
                    <path d="M5 10v10h14V10" />
                  </svg>
                </div>
                <div>
                  <h1 class="text-sm sm:text-base font-bold tracking-wide">
                    <span class="bg-gradient-to-r from-cyan-300 via-blue-400 to-purple-400 bg-clip-text text-transparent">
                      ระบบแจ้งซ่อมอุปกรณ์
                    </span>
                  </h1>
                  <p class="text-[10px] text-cyan-200/70">โรงเรียนชุมชนมาบอำมฤต</p>
                </div>
              </div>
              <div class="flex items-center gap-4">
                <div class="hidden sm:block text-right">
                  <div class="text-xs font-semibold text-white">${u.name}</div>
                  <div class="text-[10px] text-cyan-400 uppercase tracking-[0.2em]">
                    ${
                      role === "admin"
                        ? "ADMIN"
                        : role === "technician"
                        ? "TECHNICIAN"
                        : "TEACHER"
                    }
                  </div>
                </div>
                <button
                  id="btnLogout"
                  class="flex items-center gap-1.5 text-xs bg-slate-800/70 hover:bg-red-900/40 px-3 py-1.5 rounded-lg border border-slate-600 hover:border-red-500 text-slate-200 hover:text-red-300 transition-colors"
                >
                  <span>ออกจากระบบ</span>
                </button>
              </div>
            </div>
          </header>

          <!-- Content -->
          <main class="flex-1 max-w-6xl mx-auto w-full px-3 sm:px-4 py-4 sm:py-6">
            <div id="dashboardContent" class="space-y-4"></div>
          </main>
        </div>
      `;

      document.getElementById("btnLogout").onclick = () => {
        state.currentUser = null;
        render();
      };

      if (role === "teacher") {
        renderTeacherView();
      } else if (role === "technician") {
        renderTechnicianView();
      } else {
        renderAdminView();
      }
    }

    // ---------------------------
    // Teacher View (แจ้งซ่อม + ประวัติของตนเอง)
    // ---------------------------
    function renderTeacherView() {
      const container = document.getElementById("dashboardContent");
      const u = state.currentUser;

      const myName = u.name;
      const myIncidents = state.incidents
        .filter((i) => i.reporterName === myName)
        .sort((a, b) => b.timestamp - a.timestamp)
        .slice(0, 8);

      container.innerHTML = `
        <div class="grid md:grid-cols-[2fr,1.4fr] gap-4">
          <!-- Form แจ้งซ่อม -->
          <section class="glass-panel rounded-2xl p-5 space-y-4">
            <div class="flex justify-between items-center mb-2">
              <div>
                <h2 class="text-base font-bold text-white flex items-center gap-2">
                  <span class="w-7 h-7 rounded-lg bg-cyan-500/20 flex items-center justify-center text-cyan-300">
                    🛠
                  </span>
                  แบบฟอร์มแจ้งซ่อม
                </h2>
                <p class="text-[11px] text-slate-400 mt-1">กรุณากรอกข้อมูลให้ครบถ้วนในช่องที่มี * </p>
              </div>
            </div>

            <form id="reportForm" class="space-y-3">
              <div class="grid sm:grid-cols-2 gap-3">
                <div>
                  <label class="block text-xs text-slate-300 mb-1 font-semibold">อาคาร / สถานที่ *</label>
                  <select
                    id="rfBuilding"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2 text-sm text-white focus:outline-none focus:border-cyan-400"
                    required
                  >
                    <option value="">-- เลือกอาคาร --</option>
                    ${BUILDINGS.map(
                      (b) => `<option value="${b.id}">${b.name}</option>`
                    ).join("")}
                  </select>
                </div>
                <div>
                  <label class="block text-xs text-slate-300 mb-1 font-semibold">ห้อง / จุดที่พบปัญหา</label>
                  <input
                    id="rfRoom"
                    type="text"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2 text-sm text-white focus:outline-none focus:border-cyan-400"
                    placeholder="เช่น ห้อง 402, ทางเดินชั้น 2"
                  />
                </div>
              </div>

              <div>
                <label class="block text-xs text-slate-300 mb-1 font-semibold">รายละเอียดปัญหา *</label>
                <textarea
                  id="rfDesc"
                  rows="3"
                  class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2 text-sm text-white focus:outline-none focus:border-cyan-400 resize-none"
                  placeholder="ระบุอาการเสีย หรือความเสียหายที่พบ..."
                  required
                ></textarea>
              </div>

              <div class="grid sm:grid-cols-2 gap-3">
                <div>
                  <label class="block text-xs text-slate-300 mb-1 font-semibold">ระดับความเร่งด่วน</label>
                  <select
                    id="rfPriority"
                    class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2 text-sm text-white focus:outline-none focus:border-cyan-400"
                  >
                    <option value="Medium">ปานกลาง</option>
                    <option value="High">สูง</option>
                    <option value="Critical">วิกฤต</option>
                    <option value="Low">ต่ำ</option>
                  </select>
                </div>
                <div>
                  <label class="block text-xs text-slate-300 mb-1 font-semibold">ผู้แจ้ง</label>
                  <input
                    type="text"
                    class="w-full bg-slate-900/80 border border-slate-700 rounded-lg px-3 py-2 text-sm text-slate-300"
                    value="${myName}"
                    disabled
                  />
                </div>
              </div>

              <div id="rfMessage" class="text-xs text-emerald-400 h-4"></div>

              <button
                type="submit"
                class="w-full bg-emerald-600 hover:bg-emerald-500 text-white font-semibold py-2.5 rounded-lg text-sm shadow-lg transition-colors"
              >
                ส่งข้อมูลแจ้งซ่อม
              </button>
            </form>
          </section>

          <!-- ประวัติการแจ้งของฉัน -->
          <section class="glass-panel rounded-2xl p-5 flex flex-col">
            <div class="flex items-center justify-between mb-3">
              <h3 class="text-sm font-bold text-white flex items-center gap-2">
                <span class="w-7 h-7 rounded-lg bg-purple-500/20 flex items-center justify-center text-purple-300">
                  📜
                </span>
                ประวัติการแจ้งของฉัน
              </h3>
            </div>
            <div class="flex-1 overflow-y-auto custom-scrollbar space-y-2 pr-1 text-sm">
              ${
                myIncidents.length === 0
                  ? `<p class="text-xs text-slate-400 italic">ยังไม่มีประวัติการแจ้งซ่อม</p>`
                  : myIncidents
                      .map((i) => {
                        const b = BUILDINGS.find((b) => b.id === i.buildingId);
                        const statusColor =
                          i.status === "Done"
                            ? "text-emerald-400"
                            : i.status === "In Progress"
                            ? "text-blue-400"
                            : "text-amber-400";
                        const statusLabel =
                          i.status === "Done"
                            ? "เสร็จสิ้น"
                            : i.status === "In Progress"
                            ? "กำลังซ่อม"
                            : "รอจัดสรร";
                        return `
                          <div class="hud-card rounded-lg border p-3 space-y-1">
                            <div class="flex justify-between items-center">
                              <div class="text-[11px] text-slate-400">#${String(
                                i.id
                              ).padStart(4, "0")}</div>
                              <div class="text-[11px] ${statusColor} font-semibold">${statusLabel}</div>
                            </div>
                            <div class="text-sm font-semibold text-white">${i.description}</div>
                            <div class="text-[11px] text-slate-400 flex flex-wrap gap-2">
                              <span>📍 ${b ? b.name : i.buildingId}${
                          i.roomNumber ? " (" + i.roomNumber + ")" : ""
                        }</span>
                              <span>• ${formatRelativeTime(i.timestamp)}</span>
                            </div>
                          </div>
                        `;
                      })
                      .join("")
              }
            </div>
          </section>
        </div>
      `;

      document.getElementById("reportForm").addEventListener("submit", (e) => {
        e.preventDefault();
        const buildingId = document.getElementById("rfBuilding").value;
        const room = document.getElementById("rfRoom").value.trim();
        const desc = document.getElementById("rfDesc").value.trim();
        const priority = document.getElementById("rfPriority").value;

        if (!buildingId || !desc) return;

        const newIncident = {
          id: Date.now(),
          buildingId,
          buildingName: BUILDINGS.find((b) => b.id === buildingId)?.name || buildingId,
          roomNumber: room,
          description: desc,
          reporterName: myName,
          priority,
          status: "Pending",
          assignedTeamId: null,
          timestamp: Date.now(),
        };
        state.incidents.push(newIncident);
        const msgEl = document.getElementById("rfMessage");
        msgEl.textContent = "บันทึกการแจ้งซ่อมเรียบร้อยแล้ว";
        setTimeout(() => (msgEl.textContent = ""), 2500);

        document.getElementById("rfBuilding").value = "";
        document.getElementById("rfRoom").value = "";
        document.getElementById("rfDesc").value = "";
        document.getElementById("rfPriority").value = "Medium";

        renderTeacherView();
      });
    }

    // ---------------------------
    // Technician View (งานปัจจุบัน + ประวัติทีม)
    // ---------------------------
    function renderTechnicianView() {
      const container = document.getElementById("dashboardContent");
      const u = state.currentUser;
      const team = state.teams.find((t) => t.id === u.teamId);
      let currentJob = null;
      if (team && team.currentIncidentId) {
        currentJob = state.incidents.find((i) => i.id === team.currentIncidentId);
      }

      const jobHistory = state.incidents
        .filter((i) => i.assignedTeamId === team?.id && i.status === "Done")
        .sort((a, b) => b.completedAt - a.completedAt)
        .slice(0, 10);

      container.innerHTML = `
        <div class="space-y-4">
          <!-- การ์ดสถานะทีม -->
          <section class="glass-panel rounded-2xl p-4 flex items-center gap-4">
            <div class="w-12 h-12 rounded-full bg-slate-800 flex items-center justify-center border border-slate-600">
              <span class="text-cyan-300 text-xl">🔧</span>
            </div>
            <div class="flex-1">
              <div class="text-[11px] text-slate-400 uppercase tracking-[0.2em] mb-1">Technician Online</div>
              <div class="text-base font-bold">${u.name}</div>
              <div class="text-xs mt-1 flex items-center gap-2">
                <span class="px-2 py-0.5 rounded-full text-[11px] border ${
                  team?.status === "Busy"
                    ? "bg-amber-500/10 border-amber-500 text-amber-300"
                    : "bg-emerald-500/10 border-emerald-500 text-emerald-300"
                }">
                  ${
                    team?.status === "Busy"
                      ? "กำลังปฏิบัติงาน"
                      : "พร้อมรับงาน"
                  }
                </span>
                <span class="text-slate-400 text-[11px]">ทีม: ${
                  team?.name || "-"
                }</span>
              </div>
            </div>
          </section>

          <!-- งานปัจจุบัน -->
          <section class="grid md:grid-cols-[1.6fr,1.4fr] gap-4">
            <div class="glass-panel rounded-2xl p-4">
              <h2 class="text-sm font-bold mb-3 flex items-center gap-2">
                <span class="w-7 h-7 rounded-lg bg-blue-500/20 flex items-center justify-center text-blue-300">📦</span>
                งานปัจจุบัน
              </h2>
              ${
                !currentJob
                  ? `<div class="border border-dashed border-slate-700 rounded-xl p-8 text-center text-sm text-slate-400">
                      ยังไม่มีงานที่ได้รับมอบหมายในขณะนี้
                    </div>`
                  : renderTechnicianJobCard(currentJob)
              }
            </div>

            <!-- ประวัติงานทีม -->
            <div class="glass-panel rounded-2xl p-4 flex flex-col">
              <h2 class="text-sm font-bold mb-3 flex items-center gap-2">
                <span class="w-7 h-7 rounded-lg bg-purple-500/20 flex items-center justify-center text-purple-300">📜</span>
                ประวัติการปฏิบัติงาน
              </h2>
              <div class="flex-1 overflow-y-auto custom-scrollbar space-y-2 pr-1 text-sm">
                ${
                  jobHistory.length === 0
                    ? `<p class="text-xs text-slate-400 italic">ยังไม่มีประวัติงาน</p>`
                    : jobHistory
                        .map((i) => {
                          return `
                            <div class="hud-card rounded-lg border p-3 space-y-1">
                              <div class="flex justify-between items-center text-[11px] text-slate-400">
                                <span>#${String(i.id).padStart(4, "0")}</span>
                                <span>${formatRelativeTime(i.completedAt)}</span>
                              </div>
                              <div class="text-sm font-semibold text-white">${i.description}</div>
                              <div class="text-[11px] text-slate-400">
                                📍 ${i.buildingName}${i.roomNumber ? " (" + i.roomNumber + ")" : "" }
                              </div>
                            </div>
                          `;
                        })
                        .join("")
                }
              </div>
            </div>
          </section>
        </div>
      `;

      // ถ้ามีงานปัจจุบัน ให้ผูกปุ่ม "ปิดงาน"
      if (currentJob) {
        const btn = document.getElementById("btnCompleteJob");
        btn.onclick = () => {
          currentJob.status = "Done";
          currentJob.completedAt = Date.now();
          if (team) {
            team.status = "Available";
            team.currentIncidentId = null;
          }
          renderTechnicianView();
        };
      }
    }

    function renderTechnicianJobCard(job) {
      const b = BUILDINGS.find((bb) => bb.id === job.buildingId);
      return `
        <div class="rounded-xl border border-slate-700 p-4 space-y-3 bg-gradient-to-br from-slate-800 to-slate-900">
          <div class="flex justify-between items-start gap-3">
            <div>
              <div class="flex items-center gap-2 mb-1">
                <span class="text-[11px] px-2 py-0.5 rounded-full bg-cyan-500/15 text-cyan-300 border border-cyan-500/40 uppercase tracking-wide">Mission Active</span>
                <span class="text-[11px] px-2 py-0.5 rounded-full border ${
                  job.priority === "Critical"
                    ? "bg-red-500/20 border-red-400 text-red-200"
                    : "bg-blue-500/20 border-blue-400 text-blue-200"
                }">
                  ${job.priority}
                </span>
              </div>
              <div class="text-sm font-semibold text-white">${job.description}</div>
            </div>
            <div class="text-right text-[11px] text-slate-400">
              <div>JOB ID</div>
              <div class="font-mono text-base text-slate-100">#${String(job.id).padStart(4, "0")}</div>
            </div>
          </div>
          <div class="grid sm:grid-cols-2 gap-3 text-xs">
            <div class="bg-slate-900/70 rounded-lg p-2.5 border border-slate-700">
              <div class="text-[10px] text-slate-400 mb-0.5">สถานที่</div>
              <div class="text-slate-100">${b ? b.name : job.buildingId}</div>
              <div class="text-slate-400">${job.roomNumber || "-"}</div>
            </div>
            <div class="bg-slate-900/70 rounded-lg p-2.5 border border-slate-700">
              <div class="text-[10px] text-slate-400 mb-0.5">เวลาแจ้ง</div>
              <div class="text-slate-100">
                ${new Date(job.timestamp).toLocaleString("th-TH", {
                  hour: "2-digit",
                  minute: "2-digit",
                })}
              </div>
              <div class="text-slate-400">${formatRelativeTime(job.timestamp)}</div>
            </div>
          </div>
          <button
            id="btnCompleteJob"
            class="w-full mt-2 bg-emerald-600 hover:bg-emerald-500 text-white text-sm font-semibold py-2.5 rounded-lg shadow-lg"
          >
            ปิดงานนี้ (Complete)
          </button>
        </div>
      `;
    }

    // ---------------------------
    // Admin View (มองระบบภาพรวม + จัดสรรงาน)
    // ---------------------------
    function renderAdminView() {
      const container = document.getElementById("dashboardContent");

      // ตัวเลขสรุป
      const pending = state.incidents.filter((i) => i.status === "Pending").length;
      const inProgress = state.incidents.filter((i) => i.status === "In Progress").length;
      const done = state.incidents.filter((i) => i.status === "Done").length;

      const selectedBuilding = state.selectedBuildingId;
      const filters = state.filters;

      const filtered = state.incidents
        .filter((i) => {
          if (selectedBuilding && i.buildingId !== selectedBuilding) return false;
          if (filters.status !== "All" && i.status !== filters.status) return false;
          if (filters.priority !== "All" && i.priority !== filters.priority) return false;
          if (filters.search) {
            const q = filters.search.toLowerCase();
            const text =
              i.description.toLowerCase() +
              " " +
              (i.buildingName || "").toLowerCase() +
              " " +
              (i.roomNumber || "").toLowerCase() +
              " " +
              (i.reporterName || "").toLowerCase();
            if (!text.includes(q)) return false;
          }
          return true;
        })
        .sort((a, b) => b.timestamp - a.timestamp);

      container.innerHTML = `
        <div class="space-y-4">
          <!-- แถวสรุป -->
          <section class="grid sm:grid-cols-3 gap-3">
            <div class="glass-panel rounded-xl p-4">
              <div class="text-amber-300 text-2xl font-bold">${pending}</div>
              <div class="text-[11px] text-slate-300 mt-1">รอจัดสรร</div>
            </div>
            <div class="glass-panel rounded-xl p-4">
              <div class="text-blue-300 text-2xl font-bold">${inProgress}</div>
              <div class="text-[11px] text-slate-300 mt-1">กำลังซ่อม</div>
            </div>
            <div class="glass-panel rounded-xl p-4">
              <div class="text-emerald-300 text-2xl font-bold">${done}</div>
              <div class="text-[11px] text-slate-300 mt-1">เสร็จสิ้น</div>
            </div>
          </section>

          <!-- Dashboard หลัก -->
          <section class="grid lg:grid-cols-[1.2fr,1.6fr] gap-4">
            <!-- เลือกอาคาร + ทีมช่าง -->
            <div class="space-y-4">
              <!-- อาคาร -->
              <div class="glass-panel rounded-2xl p-4">
                <div class="flex justify-between items-center mb-2">
                  <h2 class="text-sm font-bold flex items-center gap-2">
                    <span class="w-7 h-7 rounded-lg bg-cyan-500/20 flex items-center justify-center text-cyan-300">🏫</span>
                    แผนผังอาคาร
                  </h2>
                  ${
                    selectedBuilding
                      ? `<button id="btnClearBuilding" class="text-[11px] text-red-400 border border-red-500/40 px-2 py-1 rounded-lg">
                          ล้างตัวกรอง
                        </button>`
                      : ""
                  }
                </div>
                <div class="space-y-2 max-h-72 overflow-y-auto custom-scrollbar pr-1">
                  ${BUILDINGS.map((b) => {
                    const count = state.incidents.filter(
                      (i) => i.buildingId === b.id && i.status !== "Done"
                    ).length;
                    const isSelected = selectedBuilding === b.id;
                    return `
                      <button
                        class="w-full text-left rounded-lg border px-3 py-2 text-sm flex justify-between items-center ${
                          isSelected
                            ? "bg-cyan-900/50 border-cyan-500 text-cyan-100"
                            : "bg-slate-900/70 border-slate-700 hover:border-cyan-400/60 hover:bg-slate-800"
                        }"
                        data-building-id="${b.id}"
                      >
                        <span>${b.name}</span>
                        ${
                          count > 0
                            ? `<span class="text-[11px] px-2 py-0.5 rounded-full bg-red-500 text-white">${count}</span>`
                            : ""
                        }
                      </button>
                    `;
                  }).join("")}
                </div>
              </div>

              <!-- ทีมช่าง -->
              <div class="glass-panel rounded-2xl p-4 space-y-2">
                <h2 class="text-sm font-bold flex items-center gap-2 mb-1">
                  <span class="w-7 h-7 rounded-lg bg-indigo-500/20 flex items-center justify-center text-indigo-300">👷</span>
                  ทีมช่าง
                </h2>
                <div class="space-y-2 text-sm">
                  ${state.teams
                    .map((t) => {
                      let badge = "";
                      if (t.status === "Busy") {
                        const job = state.incidents.find(
                          (i) => i.id === t.currentIncidentId
                        );
                        badge = `<div class="text-[11px] text-amber-300 mt-1">
                          กำลังซ่อม: ${job ? job.description : "-"}
                        </div>`;
                      }
                      return `
                        <div class="hud-card rounded-lg border p-3">
                          <div class="flex justify-between items-center">
                            <div class="font-semibold text-slate-50">${t.name}</div>
                            <div class="text-[11px] px-2 py-0.5 rounded-full border ${
                              t.status === "Busy"
                                ? "bg-amber-500/15 border-amber-400 text-amber-300"
                                : "bg-emerald-500/15 border-emerald-400 text-emerald-300"
                            }">${t.status === "Busy" ? "ไม่ว่าง" : "ว่าง"}</div>
                          </div>
                          ${badge}
                        </div>
                      `;
                    })
                    .join("")}
                </div>
              </div>
            </div>

            <!-- รายการแจ้งซ่อม -->
            <div class="glass-panel rounded-2xl p-4 flex flex-col">
              <div class="space-y-3 mb-3">
                <div class="flex gap-2">
                  <div class="relative flex-1">
                    <input
                      id="fltSearch"
                      type="text"
                      class="w-full bg-slate-950/70 border border-slate-700 rounded-lg px-3 py-2 text-sm text-white focus:outline-none focus:border-cyan-400"
                      placeholder="ค้นหา (อาคาร, ปัญหา, ผู้แจ้ง)..."
                      value="${filters.search}"
                    />
                  </div>
                  <button
                    id="btnPrint"
                    class="px-3 py-2 text-xs bg-slate-800 hover:bg-slate-700 border border-slate-600 rounded-lg"
                  >
                    พิมพ์
                  </button>
                </div>
                <div class="flex flex-wrap gap-2 text-xs">
                  <select
                    id="fltStatus"
                    class="bg-slate-950/70 border border-slate-700 rounded-lg px-2 py-1 text-[11px] text-white"
                  >
                    <option value="All"${filters.status === "All" ? " selected" : ""}>สถานะ: ทั้งหมด</option>
                    <option value="Pending"${filters.status === "Pending" ? " selected" : ""}>รอจัดสรร</option>
                    <option value="In Progress"${filters.status === "In Progress" ? " selected" : ""}>กำลังซ่อม</option>
                    <option value="Done"${filters.status === "Done" ? " selected" : ""}>เสร็จสิ้น</option>
                  </select>
                  <select
                    id="fltPriority"
                    class="bg-slate-950/70 border border-slate-700 rounded-lg px-2 py-1 text-[11px] text-white"
                  >
                    <option value="All"${filters.priority === "All" ? " selected" : ""}>ความเร่งด่วน: ทั้งหมด</option>
                    ${PRIORITIES.map(
                      (p) =>
                        `<option value="${p}"${
                          filters.priority === p ? " selected" : ""
                        }>${p}</option>`
                    ).join("")}
                  </select>
                </div>
              </div>

              <div class="flex-1 overflow-y-auto custom-scrollbar space-y-2 pr-1 text-sm">
                ${
                  filtered.length === 0
                    ? `<p class="text-xs text-slate-400 italic">ไม่พบรายการที่ตรงกับเงื่อนไข</p>`
                    : filtered
                        .map((i) => renderAdminIncidentCard(i))
                        .join("")
                }
              </div>
            </div>
          </section>
        </div>
      `;

      // handler อาคาร
      BUILDINGS.forEach((b) => {
        const btns = container.querySelectorAll(`[data-building-id="${b.id}"]`);
        btns.forEach((btn) => {
          btn.addEventListener("click", () => {
            state.selectedBuildingId =
              state.selectedBuildingId === b.id ? null : b.id;
            renderAdminView();
          });
        });
      });

      const clearBtn = document.getElementById("btnClearBuilding");
      if (clearBtn) {
        clearBtn.onclick = () => {
          state.selectedBuildingId = null;
          renderAdminView();
        };
      }

      // filter handlers
      document.getElementById("fltSearch").addEventListener("input", (e) => {
        state.filters.search = e.target.value;
        renderAdminView();
      });
      document.getElementById("fltStatus").addEventListener("change", (e) => {
        state.filters.status = e.target.value;
        renderAdminView();
      });
      document.getElementById("fltPriority").addEventListener("change", (e) => {
        state.filters.priority = e.target.value;
        renderAdminView();
      });

      document.getElementById("btnPrint").onclick = () => window.print();

      // ปุ่ม assign / complete ในแต่ละ incident
      filtered.forEach((i) => {
        const assignSelect = document.getElementById(`assignTeam-${i.id}`);
        const assignBtn = document.getElementById(`btnAssign-${i.id}`);
        const doneBtn = document.getElementById(`btnDone-${i.id}`);

        if (assignSelect && assignBtn) {
          assignBtn.onclick = () => {
            const teamId = Number(assignSelect.value);
            if (!teamId) return;
            const team = state.teams.find((t) => t.id === teamId);
            if (!team || team.status === "Busy") return;
            i.assignedTeamId = team.id;
            i.status = "In Progress";
            team.status = "Busy";
            team.currentIncidentId = i.id;
            renderAdminView();
          };
        }

        if (doneBtn) {
          doneBtn.onclick = () => {
            i.status = "Done";
            i.completedAt = Date.now();
            if (i.assignedTeamId) {
              const team = state.teams.find((t) => t.id === i.assignedTeamId);
              if (team) {
                team.status = "Available";
                team.currentIncidentId = null;
              }
            }
            renderAdminView();
          };
        }
      });
    }

    function renderAdminIncidentCard(i) {
      const isDone = i.status === "Done";
      const isPending = i.status === "Pending";
      const isInProgress = i.status === "In Progress";
      const statusColor = isDone
        ? "text-emerald-400"
        : isPending
        ? "text-amber-400"
        : "text-blue-400";
      const statusLabel = isDone
        ? "เสร็จสิ้น"
        : isPending
        ? "รอจัดสรร"
        : "กำลังซ่อม";

      const priorityBadge = (() => {
        switch (i.priority) {
          case "Critical":
            return `<span class="text-[10px] px-2 py-0.5 rounded-full bg-red-500/20 text-red-200 border border-red-400">วิกฤต</span>`;
          case "High":
            return `<span class="text-[10px] px-2 py-0.5 rounded-full bg-orange-500/20 text-orange-200 border border-orange-400">สูง</span>`;
          case "Medium":
            return `<span class="text-[10px] px-2 py-0.5 rounded-full bg-blue-500/20 text-blue-200 border border-blue-400">ปานกลาง</span>`;
          default:
            return `<span class="text-[10px] px-2 py-0.5 rounded-full bg-slate-700/30 text-slate-200 border border-slate-500">ต่ำ</span>`;
        }
      })();

      const team = i.assignedTeamId
        ? state.teams.find((t) => t.id === i.assignedTeamId)
        : null;

      const cardClasses = isDone
        ? "bg-slate-900/60 border-slate-700/80"
        : "hud-card border-slate-600/80";

      return `
        <div class="${cardClasses} rounded-lg border p-3 text-sm space-y-1">
          <div class="flex items-center justify-between gap-2">
            <div class="flex items-center gap-2">
              <span class="w-2 h-2 rounded-full ${
                isDone
                  ? "bg-emerald-400"
                  : isPending
                  ? "bg-amber-400"
                  : "bg-blue-400"
              }"></span>
              ${priorityBadge}
            </div>
            <div class="text-[11px] text-slate-400 font-mono">#${String(
              i.id
            ).padStart(4, "0")}</div>
          </div>
          <div class="font-semibold text-white">${i.description}</div>
          <div class="text-[11px] text-slate-400 flex flex-wrap gap-2">
            <span>📍 ${i.buildingName || i.buildingId}${
        i.roomNumber ? " (" + i.roomNumber + ")" : ""
      }</span>
            <span>👤 ${i.reporterName || "-"}</span>
            <span>• ${formatRelativeTime(i.timestamp)}</span>
          </div>
          <div class="flex justify-between items-center mt-1">
            <div class="text-[11px] ${statusColor} font-semibold">${statusLabel}${
        team ? " • " + team.name : ""
      }</div>
            ${
              !isDone
                ? `<div class="flex items-center gap-2">
                    <select id="assignTeam-${
                      i.id
                    }" class="text-[11px] bg-slate-950/80 border border-slate-600 rounded px-1.5 py-1 text-white">
                      <option value="">เลือกทีม</option>
                      ${state.teams
                        .map(
                          (t) =>
                            `<option value="${t.id}" ${
                              t.status === "Busy" ? "disabled" : ""
                            }>
                              ${t.name} ${t.status === "Busy" ? "(ไม่ว่าง)" : ""}
                            </option>`
                        )
                        .join("")}
                    </select>
                    <button
                      id="btnAssign-${
                        i.id
                      }"
                      class="text-[11px] bg-blue-600 hover:bg-blue-500 text-white px-2 py-1 rounded"
                    >
                      มอบหมาย
                    </button>
                    <button
                      id="btnDone-${
                        i.id
                      }"
                      class="text-[11px] bg-emerald-600 hover:bg-emerald-500 text-white px-2 py-1 rounded"
                    >
                      ปิดงาน
                    </button>
                  </div>`
                : ""
            }
          </div>
        </div>
      `;
    }

    // เริ่มต้น
    render();
  </script>
</body>
</html>
