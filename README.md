<!-- Tabs -->
<nav class="mb-4">
  <div class="flex gap-2">
    <button data-tab="dashboard" class="tab-btn px-3 py-2 rounded text-sky-700 font-medium tab-active btn">대시보드</button>
    <button data-tab="reservations" class="tab-btn px-3 py-2 rounded text-sky-700/90 btn">예약 관리</button>
    <button data-tab="devices" class="tab-btn px-3 py-2 rounded text-sky-700/90 btn">장비</button>
    <button data-tab="maintenance" class="tab-btn px-3 py-2 rounded text-sky-700/90 btn">점검·수리</button>
    <button data-tab="reports" class="tab-btn px-3 py-2 rounded text-sky-700/90 btn">보고서</button>
    <div class="ml-auto flex gap-2">
      <button id="btn-export" class="px-3 py-2 rounded bg-white text-sky-700 border border-sky-200 btn">CSV 내보내기</button>
      <button id="btn-reset" class="px-3 py-2 rounded text-red-600 border border-red-100 bg-white/60 btn">데이터 리셋</button>
    </div>
  </div>
</nav>

<!-- Content -->
<main class="space-y-6">
  <!-- Dashboard -->
  <section id="dashboard" class="tab-content">
    <div class="grid grid-cols-4 gap-4 mb-4">
      <div class="p-4 rounded-lg bg-white shadow-sm">
        <div class="text-sm text-slate-500">가용 (AVAILABLE)</div>
        <div id="cnt-available" class="text-3xl font-bold text-sky-700">0</div>
      </div>
      <div class="p-4 rounded-lg bg-white shadow-sm">
        <div class="text-sm text-slate-500">대여중 (RENTED)</div>
        <div id="cnt-rented" class="text-3xl font-bold text-sky-700">0</div>
      </div>
      <div class="p-4 rounded-lg bg-white shadow-sm">
        <div class="text-sm text-slate-500">점검 대기 (MAINTAIN)</div>
        <div id="cnt-maintain" class="text-3xl font-bold text-sky-700">0</div>
      </div>
      <div class="p-4 rounded-lg bg-white shadow-sm">
        <div class="text-sm text-slate-500">수리중 (REPAIR)</div>
        <div id="cnt-repair" class="text-3xl font-bold text-sky-700">0</div>
      </div>
    </div>

    <div class="bg-white p-4 rounded shadow-sm">
      <h2 class="font-semibold mb-2">오늘 반납 예정</h2>
      <div id="due-list" class="text-sm text-slate-700">로딩 중...</div>
    </div>
  </section>

  <!-- Reservations -->
  <section id="reservations" class="tab-content hidden">
    <div class="grid lg:grid-cols-3 gap-4">
      <div class="bg-white p-4 rounded shadow-sm">
        <h3 class="font-semibold mb-2 flex items-center gap-2">예약 생성</h3>
        <div class="space-y-2">
          <label class="block text-sm">대상자</label>
          <select id="sel-person" class="w-full border rounded p-2"></select>

          <label class="block text-sm">장비 (AVAILABLE 우선)</label>
          <select id="sel-device" class="w-full border rounded p-2"></select>

          <label class="block text-sm">시작 (날짜/시간)</label>
          <input id="inp-start" type="datetime-local" class="w-full border rounded p-2" />

          <label class="block text-sm">종료 (날짜/시간)</label>
          <input id="inp-end" type="datetime-local" class="w-full border rounded p-2" />

          <button id="btn-create-res" class="w-full mt-2 py-2 rounded bg-sky-600 text-white">예약 생성</button>
        </div>
      </div>

      <div class="lg:col-span-2 bg-white p-4 rounded shadow-sm">
        <h3 class="font-semibold mb-2">예약 / 대여 목록</h3>
        <div class="overflow-x-auto">
          <table class="w-full text-sm" id="tbl-reservations">
            <thead class="text-slate-500">
              <tr>
                <th class="p-2 text-left">ID</th>
                <th class="p-2 text-left">대상자</th>
                <th class="p-2 text-left">장비</th>
                <th class="p-2 text-left">기간</th>
                <th class="p-2 text-left">상태</th>
                <th class="p-2 text-right">작업</th>
              </tr>
            </thead>
            <tbody></tbody>
          </table>
        </div>
        <p class="text-xs text-slate-400 mt-2">픽업코드는 예약 생성 시 자동 발급됩니다. 체크인 시 코드 입력 필요.</p>
      </div>
    </div>
  </section>

  <!-- Devices -->
  <section id="devices" class="tab-content hidden">
    <div class="bg-white p-4 rounded shadow-sm">
      <h3 class="font-semibold mb-2">장비 목록</h3>
      <div class="overflow-x-auto">
        <table class="w-full text-sm" id="tbl-devices">
          <thead class="text-slate-500">
            <tr>
              <th class="p-2 text-left">장비ID</th>
              <th class="p-2 text-left">시리얼</th>
              <th class="p-2 text-left">유형</th>
              <th class="p-2 text-left">좌폭/하중</th>
              <th class="p-2 text-left">상태</th>
              <th class="p-2 text-right">작업</th>
            </tr>
          </thead>
          <tbody></tbody>
        </table>
      </div>
    </div>
  </section>

  <!-- Maintenance -->
  <section id="maintenance" class="tab-content hidden">
    <div class="bg-white p-4 rounded shadow-sm">
      <h3 class="font-semibold mb-2">점검·수리 대기</h3>
      <div id="maint-list" class="space-y-3"></div>
    </div>
  </section>

  <!-- Reports -->
  <section id="reports" class="tab-content hidden">
    <div class="bg-white p-4 rounded shadow-sm">
      <h3 class="font-semibold mb-2">간단 보고서</h3>
      <div class="grid grid-cols-3 gap-3 mb-3">
        <div class="p-3 border rounded">
          <div class="text-xs text-slate-500">총 대여건</div>
          <div id="rep-total" class="font-bold text-xl">0</div>
        </div>
        <div class="p-3 border rounded">
          <div class="text-xs text-slate-500">현재 대여중</div>
          <div id="rep-rented" class="font-bold text-xl">0</div>
        </div>
        <div class="p-3 border rounded">
          <div class="text-xs text-slate-500">점검 대기</div>
          <div id="rep-maint" class="font-bold text-xl">0</div>
        </div>
      </div>

      <div class="overflow-x-auto">
        <table class="w-full text-sm" id="tbl-report">
          <thead class="text-slate-500">
            <tr>
              <th class="p-2 text-left">ID</th>
              <th class="p-2 text-left">대상자</th>
              <th class="p-2 text-left">장비</th>
              <th class="p-2 text-left">시작</th>
              <th class="p-2 text-left">종료</th>
              <th class="p-2 text-left">상태</th>
            </tr>
          </thead>
          <tbody></tbody>
        </table>
      </div>

    </div>
  </section>

</main>

<footer class="mt-6 text-xs text-slate-500">
  ※ 데모용. 실제 배포 전에는 API 연동, 인증, 개인정보 보호, 감사로그 등을 추가하세요.
</footer>
