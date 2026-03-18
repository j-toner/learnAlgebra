<script>
    // Initial state: 3x + 6 = 12
    let equation = $state({ left: "3(x + 2)", right: "12" });
    let userMove = $state("");
    let feedback = $state({ message: "Try to solve for x.", type: "info" });
    let history = $state([]);

    function handleMove() {
        if (!userMove) return;

        // Logic Placeholder: In a real app, you'd use mathjs or a symbolic engine
        // Here is a simulation of the "Property Awareness"
        const move = userMove.toLowerCase().trim();

        if (move.includes("divide by 0")) {
            feedback = {
                message:
                    "Impossible Move: Division by zero is undefined in algebra.",
                type: "error",
            };
        } else if (move.includes("distribute")) {
            history.push({
                eq: `${equation.left} = ${equation.right}`,
                note: "Original",
            });
            equation.left = "3x + 6";
            feedback = {
                message:
                    "Applied: Distributive Property. You multiplied 3 by both x and 2.",
                type: "success",
            };
            userMove = "";
        } else if (move.includes("subtract 6")) {
            history.push({
                eq: `${equation.left} = ${equation.right}`,
                note: "Distributed",
            });
            equation.left = "3x";
            equation.right = "6";
            feedback = {
                message:
                    "Applied: Subtraction Property of Equality. Balanced both sides.",
                type: "success",
            };
            userMove = "";
        } else {
            feedback = {
                message:
                    "Move not recognized. Try 'distribute' or 'subtract 6'.",
                type: "info",
            };
        }
    }
</script>

<div class="grid gap-8 md:grid-cols-3">
    <div class="md:col-span-2 space-y-6">
        <div
            class="bg-white p-8 rounded-2xl shadow-sm border border-slate-200 text-center"
        >
            <div class="text-3xl font-mono tracking-widest mb-2">
                {equation.left} = {equation.right}
            </div>
            <p class="text-slate-400 text-sm">Current Equation</p>
        </div>

        <div class="flex gap-2">
            <input
                bind:value={userMove}
                onkeydown={(e) => e.key === "Enter" && handleMove()}
                placeholder="What move are you making? (e.g., 'distribute')"
                class="flex-1 p-3 rounded-lg border border-slate-300 focus:ring-2 focus:ring-indigo-500 outline-none"
            />
            <button
                onclick={handleMove}
                class="bg-indigo-600 text-white px-6 py-2 rounded-lg font-semibold hover:bg-indigo-700 transition"
            >
                Apply
            </button>
        </div>

        {#if feedback.message}
            <div
                class="p-4 rounded-lg flex items-start gap-3
                {feedback.type === 'error'
                    ? 'bg-red-50 text-red-700 border border-red-200'
                    : feedback.type === 'success'
                      ? 'bg-emerald-50 text-emerald-700 border border-emerald-200'
                      : 'bg-blue-50 text-blue-700 border border-blue-200'}"
            >
                <span class="font-bold"
                    >{feedback.type === "error" ? "⚠️" : "💡"}</span
                >
                {feedback.message}
            </div>
        {/if}
    </div>

    <div class="bg-slate-100 p-4 rounded-xl border border-slate-200">
        <h3 class="font-bold text-slate-700 mb-4 px-2">Solution Path</h3>
        <div class="space-y-3">
            {#each history as step}
                <div
                    class="bg-white p-3 rounded border border-slate-200 text-sm"
                >
                    <div class="font-mono text-xs text-slate-500">
                        {step.note}
                    </div>
                    <div class="font-medium">{step.eq}</div>
                </div>
            {/each}
            {#if history.length === 0}
                <p class="text-xs text-slate-400 px-2 italic">
                    Your steps will appear here...
                </p>
            {/if}
        </div>
    </div>
</div>
