<script>
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  // 페이지 타이틀 설정
  if (browser) {
    // 페이지 타이틀 설정
    document.title = "버스 출발시간";
    
    // 파비콘 설정
    const setFavicon = () => {
      const link = document.querySelector("link[rel*='icon']") || document.createElement('link');
      link.type = 'image/x-icon';
      link.rel = 'shortcut icon';
      link.href = 'data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>🚍</text></svg>';
      document.getElementsByTagName('head')[0].appendChild(link);
    };
    
    setFavicon();
  }

  // 버스 출발 시간표(14번, 700번)
  const schedules = {
    '14': [
      "06:10", "06:30", "06:45", "07:00", "07:15", "07:26", "07:37",
      "07:48", "07:59", "08:10", "08:21", "08:32", "08:43", "08:54",
      "09:05", "09:16", "09:27", "09:38", "09:49", "10:00", "10:11",
      "10:22", "10:34", "10:46", "10:58", "11:10", "11:22",
      "11:35", "11:48", "12:01", "12:14", "12:27", "12:40",
      "12:54", "13:08", "13:22", "13:36", "13:50", "14:04",
      "14:18", "14:32", "14:46", "15:00", "15:14", "15:28",
      "15:42", "15:56", "16:10", "16:24", "16:37", "16:50",
      "17:03", "17:16", "17:29", "17:42", "17:55", "18:08",
      "18:20", "18:32", "18:44", "18:56", "19:08", "19:20",
      "19:32", "19:44", "19:57", "20:10", "20:23", "20:36",
      "20:50", "21:04", "21:18", "21:32", "21:46", "22:00",
      "22:15"
    ],
    '700': [
      "06:00", "06:26", "06:52", "07:44", "08:10", "08:36", "09:02", "09:28", "10:20", "10:46",
      "11:38", "12:31", "13:28", "13:54", "14:20", "15:12", "15:38", "16:04", "16:56", "17:48",
      "18:14", "18:40", "19:40", "20:10", "20:40", "21:10", "21:40"
    ]
  };
  let selectedRoute = '14';
  $: schedule = schedules[selectedRoute];

  function getKoreaTime() {
    return new Date(new Date().toLocaleString("en-US", { timeZone: "Asia/Seoul" }));
  }

  let now = getKoreaTime();
  let nextBusTime = "";
  let nextBusInMins = 0;
  let timer;
  let showFull = false;
  let pulseTimer;
  
  // 디버깅 모드 관련 변수
  let isDebug = false;
  let debugTime = "";
  function getDebugDate() {
    let now = getKoreaTime();
    const [h, m] = debugTime.split(":").map(Number);
    return new Date(now.getFullYear(), now.getMonth(), now.getDate(), h, m, 0);
  }

  function handleTimeKeyDown(event) {
    if (event.key === "Enter") {
      updateTime();
    }
  }

  // 현재 시간과 시간표를 비교하여 다음 버스 정보를 업데이트
  function updateTime() {
    if (isDebug && debugTime) {
      now = getDebugDate();
    } else {
      now = getKoreaTime();
    }
    const currentMinutes = now.getHours() * 60 + now.getMinutes();
    const found = schedule.find(time => {
      const [h, m] = time.split(':').map(Number);
      return (h * 60 + m) >= currentMinutes;
    });
    if (found) {
      nextBusTime = found;
      const [h, m] = found.split(':').map(Number);
      const busMinutes = h * 60 + m;
      nextBusInMins = busMinutes - currentMinutes;
    } else {
      // 오늘 남은 버스가 없으면 다음 날 첫차로 계산
      nextBusTime = schedule[0];
      const [h, m] = schedule[0].split(':').map(Number);
      const busMinutes = h * 60 + m;
      nextBusInMins = (24 * 60 - currentMinutes) + busMinutes;
    }
  }

  // 간단한 펄스 효과 업데이트 (현재 사용되지 않음)
  let pulseCount = 0;
  function updatePulse() {
    pulseCount = (pulseCount + 1) % 2;
  }

  onMount(() => {
    const urlParams = new URLSearchParams(window.location.search);
    const debugParam = urlParams.get('debug');
    isDebug = debugParam === 'true';
    
    updateTime();
    timer = setInterval(updateTime, 1000);
    pulseTimer = setInterval(updatePulse, 2000);
  });

  onDestroy(() => {
    clearInterval(timer);
    clearInterval(pulseTimer);
  });

  function toggleSchedule() {
    showFull = !showFull;
  }
  
  function highlightedTimes() {
    const nowMinutes = now.getHours() * 60 + now.getMinutes();
    const timesWithDiff = schedule
      .map(time => {
        const [h, m] = time.split(':').map(Number);
        const busMinutes = h * 60 + m;
        return { time, diff: busMinutes - nowMinutes };
      })
      .filter(item => item.diff >= -2);
    timesWithDiff.sort((a, b) => Math.abs(a.diff) - Math.abs(b.diff));
    return timesWithDiff.slice(0, 3).map(item => item.time);
  }
  
  function serviceActive() {
    const nowMinutes = now.getHours() * 60 + now.getMinutes();
    const [firstH, firstM] = schedule[0].split(':').map(Number);
    const [lastH, lastM] = schedule[schedule.length - 1].split(':').map(Number);
    const firstBusMinutes = firstH * 60 + firstM;
    const lastBusMinutes = lastH * 60 + lastM;
    return (nowMinutes >= firstBusMinutes - 60) && (nowMinutes < lastBusMinutes + 60);
  }
  
  function formatBusTime(time) {
    const [h, m] = time.split(':');
    const date = new Date();
    date.setHours(Number(h));
    date.setMinutes(Number(m));
    return date.toLocaleTimeString('ko-KR', {hour: '2-digit', minute: '2-digit'});
  }
  
  function isHighlighted(time) {
    return highlightedTimes().includes(time);
  }
</script>

<main class="min-h-screen bg-gray-50 flex items-center justify-center p-4">
  <div class="w-full max-w-md animate-fade-in">
    <h1 class="text-center text-3xl font-light text-gray-800 mb-6">백석대학교 버스 출발정보</h1>
    <!-- 노선 스위치 버튼 -->
    <div class="flex justify-center mb-4">
      <button
        class="px-5 py-2 rounded-l-full border border-blue-500 font-semibold transition-all duration-150 focus:outline-none
        {selectedRoute === '14' ? 'bg-blue-500 text-white shadow-lg border-2 border-blue-700 z-10' : 'text-blue-700 bg-white hover:bg-blue-50'}"
        style="min-width: 80px;"
        on:click={() => selectedRoute = '14'}
      >14번</button>
      <button
        class="px-5 py-2 rounded-r-full border border-blue-500 font-semibold transition-all duration-150 focus:outline-none
        {selectedRoute === '700' ? 'bg-blue-500 text-white shadow-lg border-2 border-blue-700 z-10' : 'text-blue-700 bg-white hover:bg-blue-50'}"
        style="min-width: 80px;"
        on:click={() => selectedRoute = '700'}
      >700번</button>
    </div>
    
    <!-- 메인 카드 -->
    <div class="bg-white rounded-2xl shadow-lg overflow-hidden transition-all duration-300 hover:shadow-xl">
      <!-- 현재 시간 섹션 -->
      <div class="p-6 animate-slide-up">
        <div class="flex justify-between items-center">
          <span class="text-sm font-medium text-gray-500">현재 시간</span>
          <div class="flex items-end space-x-2">
            <span class="text-3xl font-bold text-gray-800">
              {now.toLocaleTimeString('ko-KR', {hour:'2-digit', minute:'2-digit'})}
            </span>
          </div>
        </div>
      </div>
      
      <!-- 구분선 -->
      <div class="h-px bg-gradient-to-r from-transparent via-gray-300 to-transparent"></div>
      
      <!-- 다음 버스 섹션 -->
      <div class="p-6 animate-slide-up" style="animation-delay: 100ms;">
        <div class="flex justify-between items-center">
          <div class="flex items-center">
            <span class="text-sm font-medium text-gray-500">다음 버스</span>
            <!-- 인디케이터 수정 -->
            <div class="relative ml-2">
              <div class="relative rounded-full h-2 w-2 bg-green-500 animate-pulse-dot"></div>
            </div>
          </div>
        {#if serviceActive()}
          <div class="flex items-center space-x-2">
            <!-- 타원형 디자인 (애니메이션 제거) -->
            <div class="w-14 h-10 rounded-full bg-green-100 flex items-center justify-center text-green-700 text-sm font-bold">
              {nextBusInMins}분 후
            </div>
            <!-- 시간 표시 -->
            <span class="text-3xl font-bold text-gray-900 whitespace-nowrap">{formatBusTime(nextBusTime)}</span>
          </div>
        {:else}
          <span class="text-3xl font-bold text-red-600">운행 종료</span>
        {/if}
        </div>
      </div>
    </div>
    
    <!-- 디버깅 모드 - isDebug가 true일 때만 표시 -->
    {#if isDebug}
      <div class="mt-6 bg-white rounded-xl p-6 shadow-md animate-slide-up" style="animation-delay: 200ms;">
        <div class="flex flex-col space-y-3">
          <div class="flex items-center justify-between">
            <label for="debug-time-input" class="text-sm font-medium text-gray-700">디버그 시간 설정</label>
            <div class="text-xs text-gray-500">현재 디버그 모드 활성화됨</div>
          </div>
          <div class="relative">
            <input type="time" 
                   id="debug-time-input"
                   bind:value={debugTime} 
                   on:change={updateTime}
                   on:keydown={handleTimeKeyDown} 
                   class="w-full py-3 px-4 text-lg border border-gray-300 rounded-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
            <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-gray-400" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm1-12a1 1 0 10-2 0v4a1 1 0 00.293.707l2.828 2.829a1 1 0 101.415-1.415L11 9.586V6z" clip-rule="evenodd" />
              </svg>
            </div>
          </div>
          <div class="flex items-center space-x-2 text-sm text-gray-500">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <span>Enter 키를 누르거나 값을 변경하면 즉시 적용됩니다</span>
          </div>
        </div>
      </div>
    {/if}
    
    <!-- 시간표 버튼 -->
    <button on:click={toggleSchedule} 
            class="mt-6 w-full bg-blue-500 hover:bg-blue-600 text-white py-4 px-6 rounded-xl font-medium transition-all duration-200 shadow hover:shadow-lg animate-slide-up flex items-center justify-center" 
            style="animation-delay: 300ms;">
      <span>{showFull ? '전체 시간표 숨기기' : '전체 시간표 보기'}</span>
      <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 ml-2" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 011.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
      </svg>
    </button>
    
    {#if showFull}
      <div class="mt-6 bg-white rounded-xl p-6 shadow-md animate-slide-up space-y-4">
        <h2 class="text-center text-lg font-medium text-gray-800">전체 시간표</h2>
        <div class="bg-gray-50 rounded-lg p-3 text-sm text-gray-600">
          현재 시간으로부터 가장 가까운 버스 3대를 강조 표시합니다.<br>
          현재 시간으로부터 최대 2분 초과된 시간까지 강조표시 됩니다.
        </div>
        <ul class="grid grid-cols-3 gap-2">
          {#each schedule as time}
            <li class="border border-gray-200 rounded-lg px-2 py-1.5 text-center text-sm text-gray-600
                       transition-all duration-200 hover:border-blue-300
                       {isHighlighted(time) ? 'bg-yellow-50 ring-2 ring-yellow-400' : ''}"
            >
              {time}
            </li>
          {/each}
        </ul>
      </div>
    {/if}

    <!-- 하단 정보 -->
    <div class="mt-8 text-center text-xs text-gray-500 animate-slide-up" style="animation-delay: 400ms;">
      <p>이 시간표는 백석대학교를 기점으로 출발하는 버스 시간을 기준으로 제작되었습니다. </p>
      <p class="mt-1">이 시간표는 실시간 버스 정보를 반영하지 않는 정적 페이지입니다.</p>
    </div>
  </div>
</main>

<style>
  :global(body) {
    font-family: 'Pretendard', sans-serif;
  }

  /* 점 펄스 애니메이션 */
  @keyframes pulse-dot {
    0% {
      transform: scale(1);
      box-shadow: 0 0 0 0 rgba(72, 187, 120, 0.7);
    }
    
    70% {
      transform: scale(1);
      box-shadow: 0 0 0 10px rgba(72, 187, 120, 0);
    }
    
    100% {
      transform: scale(1);
      box-shadow: 0 0 0 0 rgba(72, 187, 120, 0);
    }
  }

  .animate-pulse-dot {
    animation: pulse-dot 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .animate-fade-in {
    animation: fadeIn 0.5s ease-out forwards;
  }

  @keyframes slideUp {
    from {
      opacity: 0;
      transform: translateY(20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .animate-slide-up {
    animation: slideUp 0.5s ease-out forwards;
  }
</style>