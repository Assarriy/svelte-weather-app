<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import type { User } from '@supabase/supabase-js';
  import toast, { Toaster } from 'svelte-french-toast'; // Library Notifikasi

  // --- 1. TIPE DATA LENGKAP ---
  interface WeatherData {
    name: string;
    dt: number; // Time of data calculation
    main: {
      temp: number;
      humidity: number;
      pressure: number;
      feels_like: number;
    };
    weather: {
      main: string;
      description: string;
      icon: string;
    }[];
    wind: {
      speed: number;
      deg: number;
    };
    sys: {
      sunrise: number;
      sunset: number;
    };
  }

  interface ForecastItem {
    dt: number;
    dt_txt: string;
    main: {
      temp: number;
    };
    weather: {
      icon: string;
      description: string;
    }[];
  }

  interface SavedLocation {
    id: number;
    city_name: string;
    created_at: string;
  }

  // --- 2. STATE VARIABLES ---
  let city: string = '';
  let weather: WeatherData | null = null;
  let forecast: ForecastItem[] = []; // Data ramalan cuaca
  let loading: boolean = false;
  
  // State User & Data
  let user: User | null = null;
  let savedLocations: SavedLocation[] = [];

  // --- 3. HELPER FUNCTIONS (Visual & Logic) ---
  
  // Format jam (Unix timestamp -> HH:MM)
  function formatTime(timestamp: number) {
    return new Date(timestamp * 1000).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }

  // Format hari (Unix timestamp -> Senin, Selasa...)
  function formatDay(timestamp: number) {
    return new Date(timestamp * 1000).toLocaleDateString('id-ID', { weekday: 'short' });
  }

  // Menentukan Background berdasarkan cuaca utama
  function getBgColor(weatherMain: string | undefined): string {
    if (!weatherMain) return 'from-slate-900 to-slate-800';
    switch (weatherMain.toLowerCase()) {
      case 'clear': return 'from-sky-400 to-blue-600';
      case 'clouds': return 'from-slate-400 to-slate-600';
      case 'rain': 
      case 'drizzle': return 'from-indigo-800 to-slate-800';
      case 'thunderstorm': return 'from-purple-900 to-slate-900';
      case 'snow': return 'from-blue-100 to-blue-300 text-slate-800';
      default: return 'from-slate-800 to-slate-900';
    }
  }

  // --- 4. LIFECYCLE ---
  onMount(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
    });

    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
      else {
        savedLocations = [];
        weather = null;
        forecast = [];
      }
    });

    return () => subscription.unsubscribe();
  });

  // --- 5. CORE FUNCTIONS ---

  // FETCH: Helper generic untuk ambil data API
  async function fetchWeatherData(url: string, forecastUrl: string) {
    loading = true;
    try {
      // 1. Ambil Current Weather
      const res = await fetch(url);
      if (!res.ok) throw new Error('Lokasi tidak ditemukan');
      const data = await res.json();
      weather = data;

      // 2. Ambil Forecast (5 Hari)
      const resForecast = await fetch(forecastUrl);
      const dataForecast = await resForecast.json();
      
      // Filter: Ambil data jam 12:00 siang setiap hari agar tidak kebanyakan
      // atau ambil setiap 8 data (24 jam / 3 jam = 8)
      forecast = dataForecast.list.filter((item: any) => item.dt_txt.includes("12:00:00"));

      toast.success(`Cuaca di ${data.name} berhasil dimuat!`);
    } catch (err: any) {
      toast.error(err.message);
      weather = null;
      forecast = [];
    } finally {
      loading = false;
    }
  }

  // Action: Cari berdasarkan Input Text
  function handleSearch() {
    if (!city) return;
    const apiKey = import.meta.env.VITE_WEATHER_API_KEY;
    const url = `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${apiKey}&units=metric`;
    const urlForecast = `https://api.openweathermap.org/data/2.5/forecast?q=${city}&appid=${apiKey}&units=metric`;
    fetchWeatherData(url, urlForecast);
  }

  // Action: Cari berdasarkan GPS (Geolocation)
  function handleLocationClick() {
    if (!navigator.geolocation) {
      toast.error('Browser Anda tidak mendukung Geolocation');
      return;
    }
    
    loading = true;
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const { latitude, longitude } = pos.coords;
        const apiKey = import.meta.env.VITE_WEATHER_API_KEY;
        const url = `https://api.openweathermap.org/data/2.5/weather?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric`;
        const urlForecast = `https://api.openweathermap.org/data/2.5/forecast?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric`;
        fetchWeatherData(url, urlForecast);
      },
      (err) => {
        loading = false;
        toast.error('Gagal mengambil lokasi: ' + err.message);
      }
    );
  }

  // Database: Save
  async function saveLocation() {
    if (!user || !weather) {
        toast.error('Anda harus login dulu!');
        return;
    }
    
    // Cek duplikasi
    if (savedLocations.some(l => l.city_name.toLowerCase() === weather?.name.toLowerCase())) {
      toast('Kota ini sudah tersimpan', { icon: 'ℹ️' });
      return;
    }

    const { error } = await supabase.from('saved_locations').insert([{ city_name: weather.name, user_id: user.id }]);

    if (error) toast.error('Gagal menyimpan');
    else {
        toast.success('Lokasi disimpan!');
        fetchSavedLocations();
    }
  }

  // Database: Read
  async function fetchSavedLocations() {
    const { data } = await supabase.from('saved_locations').select('*').order('created_at', { ascending: false });
    if (data) savedLocations = data;
  }

  // Database: Delete
  async function deleteLocation(id: number) {
    if(!confirm("Hapus lokasi ini?")) return;
    const { error } = await supabase.from('saved_locations').delete().eq('id', id);
    if (error) toast.error('Gagal menghapus');
    else {
        savedLocations = savedLocations.filter(loc => loc.id !== id);
        toast.success('Dihapus');
    }
  }

  // Auth Functions
  const login = () => supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: `${window.location.origin}` } });
  const logout = () => { supabase.auth.signOut(); weather = null; savedLocations = []; };
  
  // Helper click list
  function loadCityFromList(cityName: string) {
    city = cityName;
    handleSearch();
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
</script>

<Toaster />

<div class={`min-h-screen text-white flex flex-col items-center py-10 px-4 transition-colors duration-1000 bg-gradient-to-br ${getBgColor(weather?.weather[0].main)}`}>
  
  <div class="absolute top-4 right-4 z-20">
    {#if user}
      <div class="flex items-center gap-3 bg-white/10 backdrop-blur-md p-2 rounded-lg border border-white/20 shadow-lg">
        <div class="text-right hidden sm:block leading-tight">
          <p class="text-[10px] uppercase opacity-70">Login as</p>
          <p class="text-xs font-bold">{user.email}</p>
        </div>
        <button on:click={logout} class="bg-red-500/80 hover:bg-red-600 text-xs px-3 py-2 rounded transition font-medium">Logout</button>
      </div>
    {:else}
      <button on:click={login} class="bg-white text-slate-900 hover:bg-gray-200 px-4 py-2 rounded-lg font-bold text-sm shadow-lg flex gap-2 items-center">
        <span>🔑</span> Login Google
      </button>
    {/if}
  </div>

  <h1 class="text-4xl md:text-5xl font-bold mb-8 mt-8 text-center drop-shadow-md">Svelte Weather Pro</h1>

  <div class="flex gap-2 mb-8 z-10 w-full max-w-md">
    <input 
      type="text" 
      bind:value={city} 
      placeholder="Cari kota..." 
      class="flex-grow p-3 rounded-xl text-slate-900 focus:outline-none shadow-xl border-0 focus:ring-4 ring-white/30"
      on:keydown={(e) => e.key === 'Enter' && handleSearch()}
    />
    <button on:click={handleSearch} class="bg-white/20 hover:bg-white/30 backdrop-blur-md px-4 py-3 rounded-xl font-bold transition shadow-xl border border-white/20">
        🔍
    </button>
    <button on:click={handleLocationClick} class="bg-emerald-500/80 hover:bg-emerald-600 px-4 py-3 rounded-xl font-bold transition shadow-xl text-white" title="Gunakan Lokasi Saya">
        📍
    </button>
  </div>

  {#if loading}
    <div class="animate-pulse flex flex-col items-center mt-10">
        <div class="w-12 h-12 border-4 border-white border-t-transparent rounded-full animate-spin mb-4"></div>
        <p>Mengunduh data langit...</p>
    </div>
  {/if}

  {#if weather && !loading}
    <div class="w-full max-w-3xl animate-fade-in-up">
        <div class="bg-white/10 backdrop-blur-lg rounded-3xl p-8 shadow-2xl border border-white/20 relative overflow-hidden">
            
            <div class="flex flex-col md:flex-row justify-between items-center gap-8">
                <div class="text-center md:text-left">
                    <h2 class="text-4xl font-bold flex items-center justify-center md:justify-start gap-2">
                        {weather.name} <span class="text-sm bg-white/20 px-2 rounded">ID</span>
                    </h2>
                    <p class="text-lg opacity-80 capitalize mt-1">{weather.weather[0].description}</p>
                    <div class="flex items-center justify-center md:justify-start mt-4">
                        <img src={`https://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`} alt="icon" class="w-24 h-24 drop-shadow-md"/>
                        <span class="text-7xl font-bold tracking-tighter">{Math.round(weather.main.temp)}°</span>
                    </div>
                </div>

                <div class="grid grid-cols-2 gap-4 text-sm w-full md:w-auto">
                    <div class="bg-black/20 p-3 rounded-xl">
                        <p class="opacity-60 text-xs uppercase">Angin</p>
                        <p class="font-bold text-lg">{weather.wind.speed} m/s</p>
                    </div>
                    <div class="bg-black/20 p-3 rounded-xl">
                        <p class="opacity-60 text-xs uppercase">Kelembapan</p>
                        <p class="font-bold text-lg">{weather.main.humidity}%</p>
                    </div>
                    <div class="bg-black/20 p-3 rounded-xl">
                        <p class="opacity-60 text-xs uppercase">Terasa Seperti</p>
                        <p class="font-bold text-lg">{Math.round(weather.main.feels_like)}°</p>
                    </div>
                    <div class="bg-black/20 p-3 rounded-xl">
                        <p class="opacity-60 text-xs uppercase">Terbit/Benam</p>
                        <p class="font-bold">{formatTime(weather.sys.sunrise)}</p>
                        <p class="font-bold">{formatTime(weather.sys.sunset)}</p>
                    </div>
                </div>
            </div>

            {#if user}
                <button on:click={saveLocation} class="mt-6 w-full bg-white/20 hover:bg-white/30 py-3 rounded-xl font-bold transition flex justify-center items-center gap-2 border border-white/10">
                    <span>💾</span> Simpan ke Favorit
                </button>
            {/if}
        </div>

        {#if forecast.length > 0}
            <h3 class="mt-8 mb-4 text-xl font-bold opacity-90 pl-2">Ramalan 5 Hari Kedepan</h3>
            <div class="grid grid-cols-2 md:grid-cols-5 gap-3">
                {#each forecast as item}
                    <div class="bg-white/10 backdrop-blur-md p-4 rounded-xl border border-white/10 text-center hover:bg-white/20 transition cursor-default">
                        <p class="text-sm font-bold opacity-70 mb-1">{formatDay(item.dt)}</p>
                        <img src={`https://openweathermap.org/img/wn/${item.weather[0].icon}.png`} alt="icon" class="w-10 h-10 mx-auto"/>
                        <p class="text-xl font-bold mt-1">{Math.round(item.main.temp)}°</p>
                        <p class="text-xs opacity-60 capitalize truncate">{item.weather[0].description}</p>
                    </div>
                {/each}
            </div>
        {/if}
    </div>
  {/if}

  {#if user && savedLocations.length > 0}
    <div class="mt-12 w-full max-w-3xl">
      <div class="flex items-center gap-4 mb-4 opacity-80">
        <h3 class="text-lg font-bold">📍 Lokasi Tersimpan</h3>
        <div class="h-[1px] bg-white/30 flex-grow"></div>
      </div>
      <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-3">
        {#each savedLocations as loc (loc.id)}
            <div class="bg-black/30 backdrop-blur-sm p-3 rounded-lg border border-white/5 flex justify-between items-center hover:bg-black/40 transition">
                <button on:click={() => loadCityFromList(loc.city_name)} class="text-left flex-grow font-semibold hover:text-sky-300 transition">
                    {loc.city_name}
                </button>
                <button on:click={() => deleteLocation(loc.id)} class="text-white/40 hover:text-red-400 p-2">✕</button>
            </div>
        {/each}
      </div>
    </div>
  {/if}
</div>