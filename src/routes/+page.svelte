<script lang="ts">
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabaseClient";
  import type { User } from "@supabase/supabase-js";

  // --- IMPORT TOAST DARI FILE SENDIRI (BUKAN LIBRARY) ---
  import Toast, { toast } from "$lib/Toast.svelte";

  import { fade, fly, slide, scale } from "svelte/transition";
  import { flip } from "svelte/animate";
  import { quintOut, elasticOut } from "svelte/easing";

  // --- ICONS ---
  const icons = {
    search:
      "M21 21l-5.197-5.197m0 0A7.5 7.5 0 105.196 5.196a7.5 7.5 0 0010.607 10.607z",
    location:
      "M15 10.5a3 3 0 11-6 0 3 3 0 016 0z M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z",
    wind: "M14.5 16h-11M17.5 12h-14M15.5 8h-9M18.5 4h-5",
    humidity:
      "M12 2.25a.75.75 0 01.75.75v2.25a.75.75 0 01-1.5 0V3a.75.75 0 01.75-.75zM7.5 12a4.5 4.5 0 119 0 4.5 4.5 0 01-9 0zM18.894 6.166a.75.75 0 00-1.06-1.06l-1.591 1.59a.75.75 0 101.06 1.061l1.591-1.59zM21.75 12a.75.75 0 01-.75.75h-2.25a.75.75 0 010-1.5h2.25a.75.75 0 01.75.75zM17.834 18.894a.75.75 0 001.06-1.06l-1.59-1.591a.75.75 0 10-1.061 1.06l1.59 1.591zM12 18a.75.75 0 01.75.75V21a.75.75 0 01-1.5 0v-2.25A.75.75 0 0112 18zM7.758 17.303a.75.75 0 00-1.061-1.06l-1.591 1.59a.75.75 0 001.06 1.061l1.591-1.59zM6 12a.75.75 0 01-.75.75H3a.75.75 0 010-1.5h2.25A.75.75 0 016 12zM6.697 7.757a.75.75 0 001.06-1.06l-1.59-1.591a.75.75 0 00-1.061 1.06l1.59 1.591z",
    sunrise:
      "M12 3v2.25m6.364.386l-1.591 1.591M21 12h-2.25m-.386 6.364l-1.591-1.591M12 18.75V21m-4.773-4.227l-1.591 1.591M5.25 12H3m4.227-4.773L5.636 5.636M12 8.25a3.75 3.75 0 100 7.5 3.75 3.75 0 000-7.5z",
    sunset: "M21 12.79A9 9 0 1111.21 3 7 7 0 0021 12.79z",
    back: "M9 15L3 9m0 0l6-6M3 9h12a6 6 0 010 12h-3",
    save: "M17.593 3.322c1.1.128 1.907 1.077 1.907 2.185V21L12 17.25 4.5 21V5.507c0-1.108.806-2.057 1.907-2.185a48.507 48.507 0 0111.186 0z",
    trash:
      "M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0",
    check: "M4.5 12.75l6 6 9-13.5",
  };

  // --- TYPES & STATE ---
  interface WeatherData {
    name: string;
    dt: number;
    main: {
      temp: number;
      humidity: number;
      feels_like: number;
      pressure: number;
    };
    weather: { main: string; description: string; icon: string }[];
    wind: { speed: number; deg: number };
    sys: { sunrise: number; sunset: number; country: string };
  }
  interface ForecastItem {
    dt: number;
    main: { temp: number };
    weather: { icon: string; description: string }[];
  }
  interface SavedLocation {
    id: number;
    city_name: string;
    created_at: string;
  }

  let city = "";
  let weather: WeatherData | null = null;
  let forecast: ForecastItem[] = [];
  let loading = false;
  let user: User | null = null;
  let savedLocations: SavedLocation[] = [];
  let showDeleteModal = false;
  let deleteTargetId: number | null = null;
  let deleteTargetName: string = "";

  $: isAlreadySaved = savedLocations.some(
    (loc) => loc.city_name.toLowerCase() === weather?.name.toLowerCase(),
  );
  const favicon = `data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>🌤️</text></svg>`;

  function sanitizeInput(input: string): string {
    return input.replace(/[^a-zA-Z0-9\s,.-]/g, "").trim();
  }
  const formatTime = (ts: number) =>
    new Date(ts * 1000).toLocaleTimeString([], {
      hour: "2-digit",
      minute: "2-digit",
    });
  const formatDay = (ts: number) =>
    new Date(ts * 1000).toLocaleDateString("id-ID", { weekday: "short" });
  function getBgColor(main: string | undefined): string {
    if (!main) return "from-[#0f172a] via-[#1e1b4b] to-[#312e81]";
    switch (main.toLowerCase()) {
      case "clear":
        return "from-blue-500 via-sky-500 to-indigo-600";
      case "clouds":
        return "from-slate-500 via-slate-600 to-zinc-700";
      case "rain":
        return "from-indigo-800 via-purple-900 to-slate-900";
      case "thunderstorm":
        return "from-gray-900 via-purple-900 to-black";
      case "drizzle":
        return "from-blue-800 via-indigo-900 to-slate-900";
      default:
        return "from-slate-800 to-gray-900";
    }
  }

  onMount(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
    });
    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_, session) => {
      user = session?.user ?? null;
      if (user) fetchSavedLocations();
      else {
        savedLocations = [];
        weather = null;
      }
    });
    return () => subscription.unsubscribe();
  });

  async function fetchWeatherData(queryUrl: string, forecastUrl: string) {
    loading = true;
    try {
      const [resW, resF] = await Promise.all([
        fetch(queryUrl),
        fetch(forecastUrl),
      ]);
      if (!resW.ok) throw new Error("Kota tidak ditemukan.");
      const dataW = await resW.json();
      const dataF = await resF.json();
      weather = dataW;
      forecast = dataF.list.filter((item: any) =>
        item.dt_txt.includes("12:00:00"),
      );
      toast.success(`Cuaca ${dataW.name} dimuat!`);
    } catch (err: any) {
      toast.error(err.message);
    } finally {
      loading = false;
    }
  }

  function handleSearch() {
    if (!city) return toast.error("Masukkan nama kota!");
    const clean = sanitizeInput(city);
    const key = import.meta.env.VITE_WEATHER_API_KEY;
    fetchWeatherData(
      `https://api.openweathermap.org/data/2.5/weather?q=${clean}&appid=${key}&units=metric`,
      `https://api.openweathermap.org/data/2.5/forecast?q=${clean}&appid=${key}&units=metric`,
    );
  }

  function handleLocation() {
    if (!navigator.geolocation) return toast.error("GPS tidak didukung");
    loading = true;
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const { latitude: lat, longitude: lon } = pos.coords;
        const key = import.meta.env.VITE_WEATHER_API_KEY;
        fetchWeatherData(
          `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${key}&units=metric`,
          `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&appid=${key}&units=metric`,
        );
      },
      () => {
        loading = false;
        toast.error("Gagal akses lokasi");
      },
    );
  }

  const resetApp = () => {
    weather = null;
    city = "";
    forecast = [];
    window.scrollTo({ top: 0, behavior: "smooth" });
  };

  async function saveLocation() {
    if (!user || !weather) return toast.error("Login diperlukan!");
    if (isAlreadySaved) return;
    const { error } = await supabase
      .from("saved_locations")
      .insert([{ city_name: weather.name, user_id: user.id }]);
    if (error) toast.error("Gagal database");
    else {
      toast.success("Disimpan!");
      fetchSavedLocations();
    }
  }

  async function fetchSavedLocations() {
    const { data } = await supabase
      .from("saved_locations")
      .select("*")
      .order("created_at", { ascending: false });
    if (data) savedLocations = data;
  }

  function requestDelete(id: number, name: string) {
    deleteTargetId = id;
    deleteTargetName = name;
    showDeleteModal = true;
  }
  async function confirmDelete() {
    if (deleteTargetId === null) return;
    const { error } = await supabase
      .from("saved_locations")
      .delete()
      .eq("id", deleteTargetId);
    if (!error) {
      savedLocations = savedLocations.filter((l) => l.id !== deleteTargetId);
      toast.success("Dihapus");
    } else {
      toast.error("Gagal menghapus");
    }
    showDeleteModal = false;
    deleteTargetId = null;
  }

  const login = () =>
    supabase.auth.signInWithOAuth({
      provider: "google",
      options: { redirectTo: window.location.origin },
    });
  const logout = () => {
    supabase.auth.signOut();
    resetApp();
    toast.success("Logout berhasil");
  };
</script>

<svelte:head>
  <title>{weather ? `Atmosphere - ${weather.name}` : "Atmosphere"}</title>
  <link rel="icon" href={favicon} />
</svelte:head>

<Toast />

{#if showDeleteModal}
  <div
    class="fixed inset-0 z-[100] flex items-center justify-center p-4"
    transition:fade={{ duration: 200 }}
  >
    <div
      class="absolute inset-0 bg-black/70 backdrop-blur-sm"
      on:click={() => (showDeleteModal = false)}
    ></div>
    <div
      class="relative bg-slate-900 border border-white/10 rounded-2xl p-6 w-full max-w-sm shadow-2xl text-center"
      in:scale={{ start: 0.95, duration: 250, easing: quintOut }}
    >
      <div
        class="w-12 h-12 bg-red-500/20 rounded-full flex items-center justify-center mx-auto mb-4 text-red-400"
      >
        <svg
          class="w-6 h-6"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
          ><path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d={icons.trash}
          /></svg
        >
      </div>
      <h3 class="text-xl font-bold text-white mb-2">Hapus Lokasi?</h3>
      <p class="text-slate-400 text-sm mb-6">
        Lokasi <span class="text-white font-semibold">"{deleteTargetName}"</span
        > akan dihapus.
      </p>
      <div class="flex gap-3 justify-center">
        <button
          on:click={() => (showDeleteModal = false)}
          class="px-5 py-2.5 rounded-xl text-slate-300 font-medium hover:bg-white/5 transition"
          >Batal</button
        >
        <button
          on:click={confirmDelete}
          class="px-5 py-2.5 rounded-xl bg-red-600 text-white font-bold hover:bg-red-700 transition shadow-lg shadow-red-900/20"
          >Hapus</button
        >
      </div>
    </div>
  </div>
{/if}

<div
  class={`min-h-screen text-white flex flex-col items-center py-6 px-4 transition-colors duration-1000 ease-in-out bg-gradient-to-br ${getBgColor(weather?.weather[0].main)} overflow-hidden relative`}
>
  <div
    class="absolute inset-0 opacity-20 pointer-events-none"
    style="background-image: url('https://grainy-gradients.vercel.app/noise.svg');"
  ></div>
  <nav class="w-full max-w-5xl flex justify-between items-center mb-10 z-50">
    <div
      class="flex items-center gap-2 cursor-pointer group"
      on:click={resetApp}
    >
      <div
        class="w-10 h-10 bg-gradient-to-tr from-cyan-400 to-blue-500 rounded-xl flex items-center justify-center shadow-lg shadow-cyan-500/30 group-hover:scale-105 transition duration-300"
      >
        <span class="text-xl">🌦️</span>
      </div>
      <span
        class="font-display font-bold text-xl tracking-tight hidden sm:block"
        >Atmosphere<span class="text-cyan-400">.</span></span
      >
    </div>
    {#if user}
      <div
        class="flex items-center gap-4"
        in:slide={{ axis: "x", duration: 400 }}
      >
        <div class="hidden sm:block text-right">
          <p class="text-[10px] uppercase opacity-60">Hello</p>
          <p class="text-sm font-semibold text-cyan-200">
            {user.email?.split("@")[0]}
          </p>
        </div>
        <button
          on:click={logout}
          class="bg-white/10 hover:bg-red-500/20 hover:text-red-300 border border-white/10 px-4 py-2 rounded-full text-xs font-bold transition"
          >Keluar</button
        >
      </div>
    {:else}
      <button
        on:click={login}
        class="bg-white text-slate-900 px-6 py-2 rounded-full text-sm font-bold shadow-lg shadow-white/10 transition hover:scale-105 hover:shadow-cyan-500/20"
        >Login</button
      >
    {/if}
  </nav>

  {#if !weather}
    <div
      class="w-full max-w-2xl flex flex-col items-center justify-center flex-grow z-10 text-center"
      in:fade={{ duration: 600 }}
    >
      <h1
        class="font-display text-5xl md:text-7xl font-bold mb-6 bg-clip-text text-transparent bg-gradient-to-b from-white to-white/50 leading-tight drop-shadow-lg"
      >
        Discover the<br />Weather
      </h1>
      <div class="w-full relative group max-w-lg mb-12">
        <div
          class="absolute -inset-1 bg-gradient-to-r from-cyan-400 to-purple-500 rounded-full blur opacity-30 group-hover:opacity-60 transition duration-1000"
        ></div>
        <div
          class="relative flex items-center bg-white/10 backdrop-blur-xl rounded-full border border-white/20 p-1.5 transition-transform group-hover:scale-[1.01]"
        >
          <input
            type="text"
            bind:value={city}
            placeholder="Cari kota (e.g. Jakarta)..."
            class="flex-grow bg-transparent border-none outline-none text-white px-6 py-3 font-medium placeholder:text-white/40"
            on:keydown={(e) => e.key === "Enter" && handleSearch()}
          />
          <button
            on:click={handleSearch}
            class="bg-white text-slate-900 p-3.5 rounded-full hover:bg-cyan-50 transition shadow-lg"
            ><svg
              class="w-5 h-5"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
              ><path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2.5"
                d={icons.search}
              /></svg
            ></button
          >
        </div>
      </div>
      <div class="flex flex-wrap justify-center gap-3">
        <button
          on:click={handleLocation}
          class="flex items-center gap-2 bg-white/5 hover:bg-white/15 border border-white/10 px-4 py-2 rounded-full text-sm transition font-medium backdrop-blur-sm"
          ><svg
            class="w-4 h-4 text-cyan-400"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d={icons.location}
            /></svg
          > Lokasi Saya</button
        >
        {#each ["Jakarta", "Bandung", "Bali", "Surabaya"] as kota}
          <button
            on:click={() => {
              city = kota;
              handleSearch();
            }}
            class="bg-white/5 hover:bg-white/15 border border-white/10 px-4 py-2 rounded-full text-sm transition text-slate-300 hover:text-white backdrop-blur-sm"
            >{kota}</button
          >
        {/each}
      </div>
    </div>
  {/if}

  {#if weather && !loading}
    <div
      class="w-full max-w-5xl z-10"
      in:fly={{ y: 50, duration: 1000, easing: elasticOut }}
    >
      <button
        on:click={resetApp}
        class="mb-6 flex items-center gap-2 text-white/60 hover:text-white transition group pl-1"
      >
        <div
          class="p-2 bg-white/5 rounded-full group-hover:bg-white/10 transition"
        >
          <svg
            class="w-5 h-5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d={icons.back}
            /></svg
          >
        </div>
        <span class="text-sm font-medium">Cari Kota Lain</span>
      </button>

      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div
          class="lg:col-span-2 glass-panel rounded-3xl p-8 md:p-10 relative overflow-hidden group"
        >
          <div
            class="absolute top-0 right-0 w-80 h-80 bg-white/5 rounded-full blur-3xl -translate-y-1/2 translate-x-1/2 group-hover:bg-white/10 transition duration-1000"
          ></div>
          <div class="flex justify-between items-start relative z-10">
            <div>
              <h2
                class="font-display text-4xl md:text-6xl font-bold mb-2 tracking-tight"
              >
                {weather.name}
              </h2>
              <p class="text-lg opacity-70 font-medium">
                {formatDay(weather.dt)}, {formatTime(weather.dt)}
              </p>
            </div>
            <div>
              <span
                class="px-3 py-1 bg-white/10 border border-white/5 rounded-lg text-sm font-bold tracking-wider"
                >{weather.sys.country}</span
              >
            </div>
          </div>
          <div
            class="flex flex-col md:flex-row items-center justify-between mt-10 md:mt-16 gap-8 relative z-10"
          >
            <div class="flex items-center gap-6">
              <img
                src={`https://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`}
                alt="weather"
                class="w-32 h-32 drop-shadow-2xl animate-pulse filter saturate-150"
              />
              <div>
                <h3
                  class="font-display text-7xl md:text-8xl font-bold tracking-tighter"
                >
                  {Math.round(weather.main.temp)}°
                </h3>
                <p class="text-xl capitalize opacity-80 font-medium">
                  {weather.weather[0].description}
                </p>
              </div>
            </div>
            {#if user}
              {#if isAlreadySaved}
                <button
                  disabled
                  class="flex items-center gap-3 bg-emerald-500/10 text-emerald-400 border border-emerald-500/20 px-8 py-4 rounded-2xl font-bold cursor-default transition"
                  ><svg
                    class="w-5 h-5"
                    fill="none"
                    stroke="currentColor"
                    viewBox="0 0 24 24"
                    stroke-width="3"
                    ><path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      d={icons.check}
                    /></svg
                  >Tersimpan</button
                >
              {:else}
                <button
                  on:click={saveLocation}
                  class="flex items-center gap-3 bg-white text-slate-900 px-8 py-4 rounded-2xl font-bold hover:scale-105 hover:shadow-xl hover:shadow-white/20 active:scale-95 transition-all duration-300 group relative overflow-hidden"
                >
                  <div
                    class="absolute inset-0 bg-gradient-to-r from-transparent via-white/50 to-transparent translate-x-[-200%] group-hover:translate-x-[200%] transition duration-700"
                  ></div>
                  <svg
                    class="w-5 h-5 group-hover:-rotate-12 transition-transform"
                    fill="none"
                    stroke="currentColor"
                    viewBox="0 0 24 24"
                    ><path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      stroke-width="2"
                      d={icons.save}
                    /></svg
                  >Simpan
                </button>
              {/if}
            {/if}
          </div>
        </div>
        <div class="grid grid-cols-2 gap-4">
          {#each [{ icon: icons.wind, label: "Angin", val: weather.wind.speed + " m/s" }, { icon: icons.humidity, label: "Kelembaban", val: weather.main.humidity + "%" }, { icon: icons.sunrise, label: "Terbit", val: formatTime(weather.sys.sunrise) }, { icon: icons.sunset, label: "Terbenam", val: formatTime(weather.sys.sunset) }] as item, i}
            <div
              in:fly={{ y: 20, delay: 200 + i * 100 }}
              class="glass-panel p-5 rounded-3xl flex flex-col items-center justify-center text-center hover:bg-white/10 transition duration-300 border-white/5"
            >
              <svg
                class="w-8 h-8 mb-3 opacity-70"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
                ><path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="1.5"
                  d={item.icon}
                /></svg
              ><span
                class="text-xs uppercase opacity-50 mb-1 font-bold tracking-wider"
                >{item.label}</span
              ><span class="text-lg font-bold">{item.val}</span>
            </div>
          {/each}
        </div>
      </div>
      {#if forecast.length > 0}
        <div class="mt-10" in:fade={{ delay: 300, duration: 800 }}>
          <h4 class="text-lg font-bold mb-5 opacity-80 flex items-center gap-3">
            <span
              class="w-1.5 h-6 bg-gradient-to-b from-cyan-400 to-blue-500 rounded-full"
            ></span> Forecast 5 Hari
          </h4>
          <div class="grid grid-cols-2 md:grid-cols-5 gap-4">
            {#each forecast as f, i}
              <div
                in:fly={{ y: 30, delay: 400 + i * 100, easing: quintOut }}
                class="glass-panel p-4 rounded-2xl flex flex-col items-center justify-center hover:bg-white/10 transition border-white/5 group hover:-translate-y-1"
              >
                <span
                  class="text-sm font-semibold opacity-60 mb-2 group-hover:text-cyan-300 transition"
                  >{formatDay(f.dt)}</span
                ><img
                  src={`https://openweathermap.org/img/wn/${f.weather[0].icon}.png`}
                  alt="icon"
                  class="w-12 h-12 mb-1 drop-shadow-lg group-hover:scale-110 transition"
                /><span class="text-xl font-bold tracking-tight"
                  >{Math.round(f.main.temp)}°</span
                >
              </div>
            {/each}
          </div>
        </div>
      {/if}
    </div>
  {/if}

  {#if loading}
    <div
      class="fixed inset-0 z-[100] bg-black/60 backdrop-blur-md flex items-center justify-center"
      transition:fade
    >
      <div
        class="flex flex-col items-center p-8 bg-black/40 rounded-3xl border border-white/10"
      >
        <div
          class="w-16 h-16 border-4 border-white/10 border-t-cyan-400 rounded-full animate-spin mb-4"
        ></div>
        <p class="text-cyan-200 animate-pulse font-medium tracking-wide">
          Menganalisis Satelit...
        </p>
      </div>
    </div>
  {/if}

  {#if user && !weather && savedLocations.length > 0}
    <div
      class="w-full max-w-4xl mt-20 z-10"
      in:slide={{ axis: "y", duration: 800, easing: quintOut }}
    >
      <div class="flex items-center gap-4 mb-6">
        <h3 class="text-xl font-display font-bold">Koleksi Favorit</h3>
        <div class="h-[1px] bg-white/10 flex-grow"></div>
        <span
          class="text-xs font-mono bg-white/10 px-2 py-1 rounded text-white/50"
          >{savedLocations.length} Kota</span
        >
      </div>
      <div class="grid grid-cols-1 sm:grid-cols-3 gap-4">
        {#each savedLocations as loc (loc.id)}
          <div
            animate:flip={{ duration: 400, easing: quintOut }}
            in:scale={{ start: 0.9, duration: 300 }}
            out:scale={{ duration: 300 }}
            class="glass-panel p-4 rounded-2xl flex justify-between items-center group hover:border-cyan-500/30 hover:bg-white/10 transition duration-300"
          >
            <button
              on:click={() => {
                city = loc.city_name;
                handleSearch();
              }}
              class="text-left"
              ><span
                class="block font-bold text-lg group-hover:text-cyan-300 transition"
                >{loc.city_name}</span
              ><span class="text-xs opacity-50"
                >{new Date(loc.created_at).toLocaleDateString()}</span
              ></button
            >
            <button
              on:click={() => requestDelete(loc.id, loc.city_name)}
              class="p-2.5 rounded-xl text-white/20 hover:text-red-400 hover:bg-red-500/10 transition"
              ><svg
                class="w-5 h-5"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
                ><path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d={icons.trash}
                /></svg
              ></button
            >
          </div>
        {/each}
      </div>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    font-family: "Inter", sans-serif;
    background-color: #0f172a;
  }
  .font-display {
    font-family: "Space Grotesk", sans-serif;
  }
  .glass-panel {
    background: rgba(255, 255, 255, 0.08);
    backdrop-filter: blur(16px);
    -webkit-backdrop-filter: blur(16px);
    border: 1px solid rgba(255, 255, 255, 0.1);
  }
</style>
