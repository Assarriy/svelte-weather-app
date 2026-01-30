<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import type { User } from '@supabase/supabase-js';
  import toast, { Toaster } from 'svelte-french-toast';
  // IMPORT TRANSISI SVELTE
  import { fade, fly } from 'svelte/transition';
  import { elasticOut } from 'svelte/easing';

  // --- TIPE DATA ---
  interface WeatherData {
    name: string;
    dt: number;
    main: { temp: number; humidity: number; feels_like: number; };
    weather: { main: string; description: string; icon: string; }[];
    wind: { speed: number; };
    sys: { sunrise: number; sunset: number; country: string; };
  }

  interface ForecastItem {
    dt: number;
    dt_txt: string;
    main: { temp: number; };
    weather: { icon: string; description: string; }[];
  }

  interface SavedLocation {
    id: number;
    city_name: string;
    created_at: string;
  }

  // --- STATE ---
  let city: string = '';
  let weather: WeatherData | null = null;
  let forecast: ForecastItem[] = [];
  let loading: boolean = false;
  let user: User | null = null;
  let savedLocations: SavedLocation[] = [];

  // --- HELPER ---
  function formatTime(timestamp: number) { return new Date(timestamp * 1000).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }); }
  function formatDay(timestamp: number) { return new Date(timestamp * 1000).toLocaleDateString('id-ID', { weekday: 'short' }); }

  // Background Dinamis yang Lebih Vibrant
  function getBgColor(weatherMain: string | undefined): string {
    if (!weatherMain) return 'from-slate-950 via-slate-900 to-blue-950';
    switch (weatherMain.toLowerCase()) {
      case 'clear': return 'from-sky-400 via-cyan-500 to-blue-600';
      case 'clouds': return 'from-slate-400 via-slate-500 to-gray-600';
      case 'rain': 
      case 'drizzle': return 'from-blue-700 via-indigo-800 to-slate-900';
      case 'thunderstorm': return 'from-purple-800 via-indigo-900 to-slate-950';
      case 'snow': return 'from-blue-200 via-blue-300 to-slate-500 text-slate-800';
      default: return 'from-slate-800 to-slate-900';
    }
  }

  // --- LIFECYCLE & AUTH ---
  onMount(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
    });
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
      else { savedLocations = []; weather = null; forecast = []; }
    });
    return () => subscription.unsubscribe();
  });

  const login = () => supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: `${window.location.origin}` } });
  const logout = () => { supabase.auth.signOut(); weather = null; savedLocations = []; toast.success('Berhasil logout'); };

  // --- CORE FUNCTIONS ---
  async function fetchWeatherData(url: string, forecastUrl: string) {
    loading = true;
    weather = null; // Reset untuk memicu animasi keluar
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error('Lokasi tidak ditemukan');
      const data = await res.json();
      
      const resForecast = await fetch(forecastUrl);
      const dataForecast = await resForecast.json();
      
      // Beri sedikit delay agar animasi loading terasa
      setTimeout(() => {
          weather = data;
          forecast = dataForecast.list.filter((item: any) => item.dt_txt.includes("12:00:00"));
          loading = false;
          toast.success(`Cuaca ${data.name} dimuat!`);
      }, 500);
      
    } catch (err: any) {
      toast.error(err.message);
      loading = false;
    }
  }

  function handleSearch() {
    if (!city) return;
    const apiKey = import.meta.env.VITE_WEATHER_API_KEY;
    fetchWeatherData(
      `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${apiKey}&units=metric`,
      `https://api.openweathermap.org/data/2.5/forecast?q=${city}&appid=${apiKey}&units=metric`
    );
  }

  function handleLocationClick() {
    if (!navigator.geolocation) { toast.error('Geolocation tidak didukung'); return; }
    loading = true;
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const { latitude, longitude } = pos.coords;
        const apiKey = import.meta.env.VITE_WEATHER_API_KEY;
        fetchWeatherData(
          `https://api.openweathermap.org/data/2.5/weather?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric`,
          `https://api.openweathermap.org/data/2.5/forecast?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric`
        );
      },
      (err) => { loading = false; toast.error('Gagal ambil lokasi: ' + err.message); }
    );
  }

  // --- DATABASE FUNCTIONS ---
  async function saveLocation() {
    if (!user || !weather) return toast.error('Login diperlukan');
    if (savedLocations.some(l => l.city_name.toLowerCase() === weather?.name.toLowerCase())) return toast('Kota sudah tersimpan', { icon: 'ℹ️' });
    const { error } = await supabase.from('saved_locations').insert([{ city_name: weather.name, user_id: user.id }]);
    if (error) toast.error('Gagal menyimpan'); else { toast.success('Tersimpan!'); fetchSavedLocations(); }
  }
  async function fetchSavedLocations() {
    const { data } = await supabase.from('saved_locations').select('*').order('created_at', { ascending: false });
    if (data) savedLocations = data;
  }
  async function deleteLocation(id: number) {
    const { error } = await supabase.from('saved_locations').delete().eq('id', id);
    if (!error) { savedLocations = savedLocations.filter(loc => loc.id !== id); toast.success('Dihapus'); }
  }
  function loadCityFromList(cityName: string) {
    city = cityName; handleSearch(); window.scrollTo({ top: 0, behavior: 'smooth' });
  }
</script>

<style>
    /* Animasi Melayang */
    @keyframes float {
        0%, 100% { transform: translateY(0px); }
        50% { transform: translateY(-15px); }
    }
    .animate-float {
        animation: float 4s ease-in-out infinite;
    }

    /* Font Poppins diterapkan ke seluruh halaman */
    :global(body) {
        font-family: 'Poppins', sans-serif;
    }

        @keyframes shimmer {
        100% { transform: translateX(100%); }
    }
    .animate-shimmer {
        animation: shimmer 1.5s infinite;
    }
</style>

<Toaster position="top-center" />

<div class={`min-h-screen text-white flex flex-col items-center py-10 px-4 transition-all duration-1000 ease-in-out bg-gradient-to-br ${getBgColor(weather?.weather[0].main)}`}>
  
  <div class="absolute top-4 right-4 z-50">
    {#if user}
      <div in:fly={{y: -20, duration: 500}} class="flex items-center gap-3 bg-white/10 backdrop-blur-md p-2 rounded-full border border-white/20 shadow-lg">
        <div class="text-right hidden sm:block leading-tight pl-2">
          <p class="text-[10px] uppercase opacity-70 tracking-wider">User</p>
          <p class="text-xs font-bold text-cyan-300">{user.email?.split('@')[0]}</p>
        </div>
        <button on:click={logout} class="bg-red-500/80 hover:bg-red-600 text-xs px-4 py-2 rounded-full transition font-bold shadow-md">Logout</button>
      </div>
    {:else}
      <button on:click={login} in:fly={{y: -20, duration: 500}} class="bg-white text-slate-900 hover:bg-cyan-50 hover:scale-105 transition-all px-6 py-2 rounded-full font-bold text-sm shadow-lg flex gap-2 items-center">
        <span>🔑</span> Login
      </button>
    {/if}
  </div>

  <h1 in:fly={{y: -50, duration: 800, delay: 200}} class="text-4xl md:text-6xl font-extrabold mb-10 mt-8 text-center drop-shadow-lg tracking-tight text-transparent bg-clip-text bg-gradient-to-r from-white via-cyan-100 to-white">
    Svelte Weather<span class="text-cyan-300">.</span>
  </h1>

  <div in:fly={{y: 20, duration: 600, delay: 400}} class="flex gap-3 mb-12 z-10 w-full max-w-lg relative group">
    <div class="absolute -inset-1 bg-gradient-to-r from-cyan-400 to-blue-500 rounded-2xl blur opacity-30 group-hover:opacity-70 transition duration-1000"></div>
    
    <input 
      type="text" 
      bind:value={city} 
      placeholder="Cari kota (contoh: Jakarta)..." 
      class="relative flex-grow p-4 rounded-2xl text-slate-900 focus:outline-none shadow-xl border-0 focus:ring-4 ring-cyan-300/50 bg-white/90 backdrop-blur-xl font-medium placeholder:text-slate-400 transition-all"
      on:keydown={(e) => e.key === 'Enter' && handleSearch()}
    />
    <button on:click={handleSearch} class="relative bg-gradient-to-br from-cyan-500 to-blue-600 hover:from-cyan-400 hover:to-blue-500 px-6 py-4 rounded-2xl font-bold transition-all shadow-xl text-2xl hover:scale-105 active:scale-95">
        🔍
    </button>
    <button on:click={handleLocationClick} class="relative bg-gradient-to-br from-emerald-500 to-teal-600 hover:from-emerald-400 hover:to-teal-500 px-6 py-4 rounded-2xl font-bold transition-all shadow-xl text-2xl hover:scale-105 active:scale-95 text-white" title="Gunakan Lokasi Saya">
        📍
    </button>
  </div>

  {#if loading}
    <div in:fade class="flex flex-col items-center mt-10 z-20">
        <div class="relative w-20 h-20">
            <div class="absolute top-0 left-0 w-full h-full border-8 border-white/20 rounded-full"></div>
            <div class="absolute top-0 left-0 w-full h-full border-8 border-cyan-400 border-t-transparent rounded-full animate-spin"></div>
        </div>
        <p class="mt-6 font-semibold text-lg animate-pulse tracking-wider drop-shadow">Memindai Atmosfer...</p>
    </div>
  {/if}

  {#if weather && !loading}
    <div class="w-full max-w-4xl" in:fly="{{ y: 50, duration: 1000, easing: elasticOut }}">
        
        <div class="bg-white/10 backdrop-blur-xl rounded-[2.5rem] p-8 md:p-12 shadow-2xl border-2 border-white/30 relative overflow-hidden group transition-all hover:border-white/50">
            
            <div class="absolute top-0 left-0 w-full h-full bg-gradient-to-br from-white/10 to-transparent opacity-0 group-hover:opacity-100 transition duration-700 pointer-events-none"></div>

            <div class="flex flex-col md:flex-row justify-between items-center gap-10 relative z-10">
                <div class="text-center md:text-left flex-1">
                    <div class="inline-block bg-black/20 px-4 py-1 rounded-full text-sm font-bold mb-2 shadow-inner">
                        {formatDay(weather.dt)}, {formatTime(weather.dt)}
                    </div>
                    <h2 class="text-5xl md:text-6xl font-extrabold flex items-center justify-center md:justify-start gap-3 drop-shadow-lg leading-none my-4">
                        {weather.name} 
                        <span class="text-lg bg-cyan-500/80 px-3 py-1 rounded-full shadow-lg">{weather.sys.country}</span>
                    </h2>
                    <p class="text-2xl font-medium opacity-90 capitalize tracking-wide mb-6 bg-white/10 inline-block px-4 py-1 rounded-lg">{weather.weather[0].description}</p>
                    
                    {#if user}
                        <button on:click={saveLocation} class="group/btn relative overflow-hidden bg-gradient-to-r from-emerald-500 to-teal-600 hover:from-emerald-400 hover:to-teal-500 text-white font-bold py-4 px-8 rounded-2xl transition-all shadow-xl shadow-emerald-900/30 hover:shadow-emerald-500/50 hover:-translate-y-1 active:translate-y-0 flex items-center gap-3 mx-auto md:mx-0">
                            <span class="text-2xl group-hover/btn:rotate-12 transition">💾</span> 
                            <span>Simpan Lokasi Ini</span>
                            <div class="absolute inset-0 h-full w-full bg-white/20 skew-x-12 -translate-x-64 group-hover/btn:animate-shimmer"></div>
                        </button>
                    {/if}
                </div>

                <div class="flex flex-col items-center justify-center md:items-end">
                    <img 
                        src={`https://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`} 
                        alt="icon" 
                        class="w-48 h-48 drop-shadow-2xl animate-float filter saturate-150"
                    />
                    <span class="text-8xl md:text-9xl font-extrabold tracking-tighter drop-shadow-xl bg-clip-text text-transparent bg-gradient-to-b from-white to-cyan-200 leading-none -mt-8">
                        {Math.round(weather.main.temp)}°
                    </span>
                    <p class="text-lg opacity-80 mt-2 font-semibold">Terasa {Math.round(weather.main.feels_like)}°</p>
                </div>
            </div>

            <div class="grid grid-cols-2 md:grid-cols-4 gap-4 mt-10 relative z-10">
                <div class="bg-black/20 hover:bg-black/30 p-4 rounded-2xl text-center transition-all hover:scale-105 border border-white/5">
                    <p class="opacity-60 text-xs font-bold uppercase tracking-wider mb-2">Kecepatan Angin</p>
                    <p class="font-extrabold text-2xl">{weather.wind.speed} <span class="text-sm">m/s</span></p>
                </div>
                <div class="bg-black/20 hover:bg-black/30 p-4 rounded-2xl text-center transition-all hover:scale-105 border border-white/5">
                    <p class="opacity-60 text-xs font-bold uppercase tracking-wider mb-2">Kelembapan</p>
                    <p class="font-extrabold text-2xl">{weather.main.humidity}<span class="text-sm">%</span></p>
                </div>
                <div class="bg-black/20 hover:bg-black/30 p-4 rounded-2xl text-center transition-all hover:scale-105 border border-white/5">
                    <p class="opacity-60 text-xs font-bold uppercase tracking-wider mb-2">Matahari Terbit</p>
                    <p class="font-extrabold text-lg">🌅 {formatTime(weather.sys.sunrise)}</p>
                </div>
                <div class="bg-black/20 hover:bg-black/30 p-4 rounded-2xl text-center transition-all hover:scale-105 border border-white/5">
                    <p class="opacity-60 text-xs font-bold uppercase tracking-wider mb-2">Matahari Terbenam</p>
                    <p class="font-extrabold text-lg">🌇 {formatTime(weather.sys.sunset)}</p>
                </div>
            </div>
        </div>

        {#if forecast.length > 0}
            <div class="mt-10">
                <h3 class="text-xl font-bold opacity-90 mb-6 pl-4 border-l-4 border-cyan-400 drop-shadow-md">Forecast 5 Hari Kedepan</h3>
                <div class="grid grid-cols-2 md:grid-cols-5 gap-4">
                    {#each forecast as item, i}
                        <div in:fly={{ y: 50, duration: 800, delay: i * 150, easing: elasticOut }} 
                             class="bg-white/10 backdrop-blur-md p-6 rounded-3xl border-2 border-white/20 text-center hover:bg-white/20 transition-all hover:-translate-y-2 hover:shadow-xl hover:border-cyan-300/50 group cursor-default">
                            <p class="text-sm font-extrabold opacity-70 mb-2">{formatDay(item.dt)}</p>
                            <img src={`https://openweathermap.org/img/wn/${item.weather[0].icon}.png`} alt="icon" class="w-16 h-16 mx-auto drop-shadow-lg group-hover:scale-110 transition"/>
                            <p class="text-3xl font-extrabold mt-2 mb-1">{Math.round(item.main.temp)}°</p>
                            <p class="text-xs font-bold opacity-60 capitalize truncate bg-black/20 py-1 px-2 rounded-full mx-auto inline-block">{item.weather[0].description}</p>
                        </div>
                    {/each}
                </div>
            </div>
        {/if}
    </div>
  {/if}

  {#if user && savedLocations.length > 0 && !loading && !weather}
    <div in:fly={{y: 50, duration: 800}} class="mt-8 w-full max-w-3xl">
      <div class="flex items-center gap-4 mb-8 opacity-90">
        <h3 class="text-2xl font-extrabold drop-shadow">📍 Lokasi Favorit Anda</h3>
        <div class="h-[2px] bg-gradient-to-r from-white/50 to-transparent flex-grow rounded-full"></div>
      </div>
      <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
        {#each savedLocations as loc, i (loc.id)}
            <div in:fly={{ x: -50, duration: 600, delay: i * 100 }}
                 class="group bg-gradient-to-r from-white/10 to-white/5 backdrop-blur-md p-5 rounded-2xl border-2 border-white/10 flex justify-between items-center hover:border-cyan-400/50 hover:shadow-lg hover:shadow-cyan-500/20 transition-all hover:-translate-x-1">
                <button on:click={() => loadCityFromList(loc.city_name)} class="text-left flex-grow">
                    <span class="block font-extrabold text-xl group-hover:text-cyan-300 transition">{loc.city_name}</span>
                    <span class="text-xs opacity-60 font-medium">Disimpan pada {new Date(loc.created_at).toLocaleDateString('id-ID')}</span>
                </button>
                <button on:click={() => deleteLocation(loc.id)} class="bg-white/10 text-white/60 hover:text-red-200 hover:bg-red-500/80 p-3 rounded-xl transition-all hover:rotate-90 active:scale-90 shadow-sm">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" /></svg>
                </button>
            </div>
        {/each}
      </div>
    </div>
  {/if}
</div>