<script>
    import "./layout.css";
    let { children } = $props();
</script>

<div class="min-h-screen bg-slate-50 text-slate-900 font-sans">
    <nav class="border-b bg-white p-4">
        <div class="max-w-4xl mx-auto flex justify-between items-center">
            <h1 class="text-xl font-bold tracking-tight text-indigo-600">
                AlgebraStep <span class="text-slate-400 font-light">| Lab</span>
            </h1>
            <div class="text-sm text-slate-500">Practice Mode</div>
        </div>
    </nav>

    <main class="max-w-4xl mx-auto p-6">
        {@render children()}
    </main>
</div>
