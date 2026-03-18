<script lang="ts">
    // ─── Types ───────────────────────────────────────────────────────────────────

    type Term = { coeff: number; variable: string | null };
    type Side = Term[];

    interface Equation {
        left: Side;
        right: Side;
    }

    interface Step {
        equation: Equation;
        property: string;
        description: string;
    }

    type FeedbackKind = "property" | "error" | "solved";

    interface Feedback {
        kind: FeedbackKind;
        title: string;
        body: string;
    }

    // ─── Equation helpers ─────────────────────────────────────────────────────────

    function cloneSide(s: Side): Side {
        return s.map((t) => ({ ...t }));
    }

    function cloneEq(eq: Equation): Equation {
        return { left: cloneSide(eq.left), right: cloneSide(eq.right) };
    }

    function simplify(side: Side): Side {
        const vars = new Map<string | null, number>();
        for (const t of side) {
            vars.set(t.variable, (vars.get(t.variable) ?? 0) + t.coeff);
        }
        const result: Side = [];
        for (const [variable, coeff] of vars) {
            if (coeff !== 0) result.push({ coeff, variable });
        }
        if (result.length === 0) result.push({ coeff: 0, variable: null });
        return result;
    }

    function sideEquals(a: Side, b: Side): boolean {
        const sa = simplify(a);
        const sb = simplify(b);
        if (sa.length !== sb.length) return false;
        for (const ta of sa) {
            const tb = sb.find((t) => t.variable === ta.variable);
            if (!tb || tb.coeff !== ta.coeff) return false;
        }
        return true;
    }

    function formatSide(side: Side): string {
        const s = simplify(side);
        if (s.length === 0) return "0";
        return s
            .map((t, i) => {
                const isFirst = i === 0;
                const abs = Math.abs(t.coeff);
                const sign =
                    t.coeff < 0
                        ? isFirst
                            ? "−"
                            : " − "
                        : isFirst
                          ? ""
                          : " + ";
                if (t.variable === null) return sign + abs;
                if (abs === 1) return sign + t.variable;
                return sign + abs + t.variable;
            })
            .join("");
    }

    function isSolved(eq: Equation): boolean {
        const ls = simplify(eq.left);
        const rs = simplify(eq.right);
        const leftIsVar =
            ls.length === 1 && ls[0].variable !== null && ls[0].coeff === 1;
        const rightIsConst = rs.length === 1 && rs[0].variable === null;
        const leftIsConst = ls.length === 1 && ls[0].variable === null;
        const rightIsVar =
            rs.length === 1 && rs[0].variable !== null && rs[0].coeff === 1;
        return (leftIsVar && rightIsConst) || (leftIsConst && rightIsVar);
    }

    // ─── Operation logic ──────────────────────────────────────────────────────────

    type Operation = "add" | "subtract" | "multiply" | "divide" | "simplify";

    interface MoveResult {
        ok: boolean;
        equation?: Equation;
        property?: string;
        description?: string;
        errorTitle?: string;
        errorBody?: string;
    }

    function applyMove(
        eq: Equation,
        op: Operation,
        value: number | null,
    ): MoveResult {
        const next = cloneEq(eq);

        if (op === "simplify") {
            const newLeft = simplify(next.left);
            const newRight = simplify(next.right);
            const alreadySimple =
                sideEquals(newLeft, next.left) &&
                sideEquals(newRight, next.right) &&
                next.left.length === newLeft.length &&
                next.right.length === newRight.length;

            if (alreadySimple) {
                return {
                    ok: false,
                    errorTitle: "Nothing to simplify",
                    errorBody:
                        "Both sides are already fully simplified — no like terms to combine.",
                };
            }

            next.left = newLeft;
            next.right = newRight;
            return {
                ok: true,
                equation: next,
                property: "Combining Like Terms",
                description:
                    "We grouped terms with the same variable and added their coefficients, reducing the expression to its simplest form.",
            };
        }

        if (value === null || isNaN(value)) {
            return {
                ok: false,
                errorTitle: "Enter a value",
                errorBody: "Please type a number to use with this operation.",
            };
        }

        if (op === "add") {
            next.left.push({ coeff: value, variable: null });
            next.right.push({ coeff: value, variable: null });
            return {
                ok: true,
                equation: next,
                property: "Addition Property of Equality",
                description: `Adding ${value} to both sides keeps the equation balanced — whatever you do to one side, you must do to the other.`,
            };
        }

        if (op === "subtract") {
            next.left.push({ coeff: -value, variable: null });
            next.right.push({ coeff: -value, variable: null });
            return {
                ok: true,
                equation: next,
                property: "Subtraction Property of Equality",
                description: `Subtracting ${value} from both sides keeps the equation balanced. This is the same as adding −${value}.`,
            };
        }

        if (op === "multiply") {
            if (value === 0) {
                return {
                    ok: false,
                    errorTitle: "Can't multiply by zero",
                    errorBody:
                        "Multiplying both sides by 0 would make everything 0 = 0, destroying all information about the variable. This move is not allowed.",
                };
            }
            next.left = next.left.map((t) => ({
                ...t,
                coeff: t.coeff * value,
            }));
            next.right = next.right.map((t) => ({
                ...t,
                coeff: t.coeff * value,
            }));
            return {
                ok: true,
                equation: next,
                property: "Multiplication Property of Equality",
                description: `Multiplying both sides by ${value} scales the equation evenly. The equality is preserved because both sides change by the same factor.`,
            };
        }

        if (op === "divide") {
            if (value === 0) {
                return {
                    ok: false,
                    errorTitle: "Division by zero is undefined",
                    errorBody:
                        "You can't divide by zero — it's undefined in mathematics. No number multiplied by 0 gives a non-zero result, so the operation has no meaning.",
                };
            }
            next.left = next.left.map((t) => ({
                ...t,
                coeff: t.coeff / value,
            }));
            next.right = next.right.map((t) => ({
                ...t,
                coeff: t.coeff / value,
            }));
            return {
                ok: true,
                equation: next,
                property: "Division Property of Equality",
                description: `Dividing both sides by ${value} is valid and keeps equality, though it may introduce fractional coefficients.`,
            };
        }

        return {
            ok: false,
            errorTitle: "Unknown operation",
            errorBody: "That operation is not recognized.",
        };
    }

    // ─── Problem bank ─────────────────────────────────────────────────────────────

    interface Problem {
        label: string;
        equation: Equation;
    }

    const PROBLEMS: Problem[] = [
        {
            label: "2x + 3 = 11",
            equation: {
                left: [
                    { coeff: 2, variable: "x" },
                    { coeff: 3, variable: null },
                ],
                right: [{ coeff: 11, variable: null }],
            },
        },
        {
            label: "3x − 5 = 16",
            equation: {
                left: [
                    { coeff: 3, variable: "x" },
                    { coeff: -5, variable: null },
                ],
                right: [{ coeff: 16, variable: null }],
            },
        },
        {
            label: "4x + 2 = 2x + 10",
            equation: {
                left: [
                    { coeff: 4, variable: "x" },
                    { coeff: 2, variable: null },
                ],
                right: [
                    { coeff: 2, variable: "x" },
                    { coeff: 10, variable: null },
                ],
            },
        },
        {
            label: "5x = 35",
            equation: {
                left: [{ coeff: 5, variable: "x" }],
                right: [{ coeff: 35, variable: null }],
            },
        },
        {
            label: "−3x + 9 = 0",
            equation: {
                left: [
                    { coeff: -3, variable: "x" },
                    { coeff: 9, variable: null },
                ],
                right: [{ coeff: 0, variable: null }],
            },
        },
    ];

    // ─── State ────────────────────────────────────────────────────────────────────

    let problemIndex = $state(0);
    let current = $state<Equation>(cloneEq(PROBLEMS[0].equation));
    let history = $state<Step[]>([]);
    let feedback = $state<Feedback | null>(null);
    let inputValue = $state("");
    let selectedOp = $state<Operation>("add");
    let solved = $state(false);
    let feedbackVisible = $state(false);

    // ─── Derived ──────────────────────────────────────────────────────────────────

    let parsedValue = $derived(parseFloat(inputValue));
    let needsValue = $derived(selectedOp !== "simplify");

    // ─── Actions ──────────────────────────────────────────────────────────────────

    function applyOperation() {
        if (solved) return;

        const result = applyMove(
            current,
            selectedOp,
            needsValue ? parsedValue : null,
        );

        if (!result.ok) {
            feedback = {
                kind: "error",
                title: result.errorTitle!,
                body: result.errorBody!,
            };
            triggerFeedback();
            return;
        }

        history = [
            ...history,
            {
                equation: cloneEq(current),
                property: result.property!,
                description: result.description!,
            },
        ];

        current = result.equation!;

        feedback = {
            kind: "property",
            title: result.property!,
            body: result.description!,
        };
        triggerFeedback();

        if (isSolved(result.equation!)) {
            solved = true;
            feedback = {
                kind: "solved",
                title: "🎉 Solved!",
                body: `You solved the equation in ${history.length} step${history.length === 1 ? "" : "s"}. Well done!`,
            };
        }

        inputValue = "";
    }

    function triggerFeedback() {
        feedbackVisible = false;
        setTimeout(() => {
            feedbackVisible = true;
        }, 10);
    }

    function loadProblem(index: number) {
        problemIndex = index;
        current = cloneEq(PROBLEMS[index].equation);
        history = [];
        feedback = null;
        feedbackVisible = false;
        solved = false;
        inputValue = "";
    }

    function undoLastStep() {
        if (history.length === 0 || solved) return;
        const prev = history[history.length - 1];
        current = cloneEq(prev.equation);
        history = history.slice(0, -1);
        feedback = null;
        feedbackVisible = false;
    }

    function handleKeydown(e: KeyboardEvent) {
        if (e.key === "Enter") applyOperation();
    }

    const OPERATIONS: { op: Operation; label: string; symbol: string }[] = [
        { op: "add", label: "Add", symbol: "+" },
        { op: "subtract", label: "Subtract", symbol: "−" },
        { op: "multiply", label: "Multiply", symbol: "×" },
        { op: "divide", label: "Divide", symbol: "÷" },
        { op: "simplify", label: "Simplify", symbol: "∑" },
    ];
</script>

<main>
    <header>
        <div class="header-inner">
            <span class="logo">∫ Algebraica</span>
            <span class="subtitle">Step-by-step equation solver</span>
        </div>
    </header>

    <div class="layout">
        <aside>
            <h2 class="sidebar-title">Problems</h2>
            <ul class="problem-list">
                {#each PROBLEMS as prob, i}
                    <li>
                        <button
                            class="problem-btn"
                            class:active={i === problemIndex}
                            onclick={() => loadProblem(i)}
                        >
                            {prob.label}
                        </button>
                    </li>
                {/each}
            </ul>
        </aside>

        <section class="workspace">
            <div class="equation-card">
                <div class="equation-label">Current equation</div>
                <div class="equation-display" class:solved>
                    <span class="side">{formatSide(current.left)}</span>
                    <span class="eq-sign">=</span>
                    <span class="side">{formatSide(current.right)}</span>
                </div>
            </div>

            {#if feedbackVisible && feedback}
                <div class="feedback {feedback.kind}" role="alert">
                    <strong class="feedback-title">{feedback.title}</strong>
                    <p class="feedback-body">{feedback.body}</p>
                </div>
            {/if}

            {#if !solved}
                <div class="controls-card">
                    <div class="op-row">
                        {#each OPERATIONS as { op, label, symbol }}
                            <button
                                class="op-btn"
                                class:selected={selectedOp === op}
                                onclick={() => {
                                    selectedOp = op;
                                }}
                                title={label}
                            >
                                <span class="op-symbol">{symbol}</span>
                                <span class="op-label">{label}</span>
                            </button>
                        {/each}
                    </div>

                    <div class="action-row">
                        {#if needsValue}
                            <input
                                type="number"
                                class="value-input"
                                bind:value={inputValue}
                                placeholder="Enter a number…"
                                onkeydown={handleKeydown}
                            />
                        {/if}
                        <button class="apply-btn" onclick={applyOperation}
                            >Apply</button
                        >
                        {#if history.length > 0}
                            <button class="undo-btn" onclick={undoLastStep}
                                >↩ Undo</button
                            >
                        {/if}
                    </div>
                </div>
            {:else}
                <div class="solved-actions">
                    <button
                        class="next-btn"
                        onclick={() =>
                            loadProblem((problemIndex + 1) % PROBLEMS.length)}
                    >
                        Next problem →
                    </button>
                    <button
                        class="retry-btn"
                        onclick={() => loadProblem(problemIndex)}
                        >Try again</button
                    >
                </div>
            {/if}

            {#if history.length > 0}
                <div class="history">
                    <h3 class="history-title">Your steps</h3>
                    <ol class="step-list">
                        {#each history as step, i}
                            <li class="step">
                                <div class="step-eq">
                                    {formatSide(step.equation.left)} = {formatSide(
                                        step.equation.right,
                                    )}
                                </div>
                                <div class="step-meta">
                                    <span class="step-num">Step {i + 1}</span>
                                    <span class="step-property"
                                        >{step.property}</span
                                    >
                                </div>
                            </li>
                        {/each}
                    </ol>
                </div>
            {/if}
        </section>
    </div>
</main>

<style>
    main {
        min-height: 100vh;
        display: flex;
        flex-direction: column;
    }

    header {
        background: var(--bg-surface);
        border-bottom: 1px solid rgba(255, 255, 255, 0.07);
        padding: 1rem 2rem;
    }

    .header-inner {
        max-width: 1100px;
        margin: 0 auto;
        display: flex;
        align-items: baseline;
        gap: 1rem;
    }

    .logo {
        font-family: var(--font-display);
        font-size: 1.5rem;
        font-weight: 700;
        color: var(--accent-gold);
        letter-spacing: -0.02em;
    }

    .subtitle {
        font-size: 0.8rem;
        color: var(--chalk-dim);
        font-family: var(--font-mono);
    }

    .layout {
        flex: 1;
        display: grid;
        grid-template-columns: 220px 1fr;
        max-width: 1100px;
        margin: 0 auto;
        width: 100%;
        padding: 2rem;
        gap: 2rem;
        align-items: start;
    }

    aside {
        background: var(--bg-surface);
        border-radius: var(--radius);
        padding: 1.25rem;
        border: 1px solid rgba(255, 255, 255, 0.06);
    }

    .sidebar-title {
        font-family: var(--font-mono);
        font-size: 0.7rem;
        text-transform: uppercase;
        letter-spacing: 0.12em;
        color: var(--chalk-dim);
        margin-bottom: 0.75rem;
    }

    .problem-list {
        list-style: none;
        display: flex;
        flex-direction: column;
        gap: 0.35rem;
    }

    .problem-btn {
        width: 100%;
        text-align: left;
        background: transparent;
        border: 1px solid transparent;
        border-radius: var(--radius);
        padding: 0.55rem 0.75rem;
        cursor: pointer;
        color: var(--chalk-dim);
        font-family: var(--font-mono);
        font-size: 0.85rem;
        transition: all 0.15s;
    }

    .problem-btn:hover {
        background: var(--bg-raised);
        color: var(--chalk);
        border-color: rgba(255, 255, 255, 0.1);
    }

    .problem-btn.active {
        background: rgba(240, 192, 96, 0.12);
        border-color: var(--accent-gold);
        color: var(--accent-gold);
    }

    .workspace {
        display: flex;
        flex-direction: column;
        gap: 1.25rem;
    }

    .equation-card {
        background: var(--bg-surface);
        border: 1px solid rgba(255, 255, 255, 0.08);
        border-radius: var(--radius);
        padding: 2rem 2.5rem;
        text-align: center;
    }

    .equation-label {
        font-family: var(--font-mono);
        font-size: 0.7rem;
        text-transform: uppercase;
        letter-spacing: 0.1em;
        color: var(--chalk-dim);
        margin-bottom: 1.25rem;
    }

    .equation-display {
        font-family: var(--font-display);
        font-size: 2.8rem;
        color: var(--chalk);
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 1.25rem;
        transition: color 0.4s;
    }

    .equation-display.solved {
        color: var(--accent-green);
    }

    .eq-sign {
        color: var(--accent-teal);
        font-weight: 400;
    }

    .feedback {
        border-radius: var(--radius);
        padding: 1rem 1.25rem;
        border-left: 4px solid;
        animation: slide-in 0.25s ease-out;
    }

    @keyframes slide-in {
        from {
            opacity: 0;
            transform: translateY(-6px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .feedback.property {
        background: rgba(94, 196, 176, 0.1);
        border-color: var(--accent-teal);
    }

    .feedback.error {
        background: rgba(224, 90, 90, 0.1);
        border-color: var(--accent-red);
    }

    .feedback.solved {
        background: rgba(94, 196, 122, 0.12);
        border-color: var(--accent-green);
    }

    .feedback-title {
        display: block;
        font-family: var(--font-mono);
        font-size: 0.8rem;
        font-weight: 600;
        letter-spacing: 0.04em;
        margin-bottom: 0.35rem;
    }

    .feedback.property .feedback-title {
        color: var(--accent-teal);
    }
    .feedback.error .feedback-title {
        color: var(--accent-red);
    }
    .feedback.solved .feedback-title {
        color: var(--accent-green);
    }

    .feedback-body {
        font-size: 0.9rem;
        color: var(--chalk-dim);
        line-height: 1.55;
    }

    .controls-card {
        background: var(--bg-surface);
        border: 1px solid rgba(255, 255, 255, 0.08);
        border-radius: var(--radius);
        padding: 1.25rem;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .op-row {
        display: flex;
        gap: 0.5rem;
    }

    .op-btn {
        flex: 1;
        background: var(--bg);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: var(--radius);
        padding: 0.6rem 0.4rem;
        cursor: pointer;
        color: var(--chalk-dim);
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.2rem;
        transition: all 0.15s;
    }

    .op-btn:hover {
        background: var(--bg-raised);
        color: var(--chalk);
        border-color: rgba(255, 255, 255, 0.2);
    }

    .op-btn.selected {
        background: rgba(240, 192, 96, 0.12);
        border-color: var(--accent-gold);
        color: var(--accent-gold);
    }

    .op-symbol {
        font-size: 1.2rem;
        font-family: var(--font-display);
    }

    .op-label {
        font-size: 0.65rem;
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.08em;
    }

    .action-row {
        display: flex;
        gap: 0.75rem;
        align-items: center;
    }

    .value-input {
        flex: 1;
        background: var(--bg);
        border: 1px solid rgba(255, 255, 255, 0.12);
        border-radius: var(--radius);
        padding: 0.65rem 1rem;
        color: var(--chalk);
        font-family: var(--font-mono);
        font-size: 1rem;
        outline: none;
        transition: border-color 0.15s;
    }

    .value-input:focus {
        border-color: var(--accent-gold);
    }

    .value-input::placeholder {
        color: var(--chalk-dim);
    }

    .apply-btn {
        background: var(--accent-gold);
        color: var(--bg);
        border: none;
        border-radius: var(--radius);
        padding: 0.65rem 1.5rem;
        font-family: var(--font-mono);
        font-weight: 600;
        font-size: 0.85rem;
        cursor: pointer;
        transition: opacity 0.15s;
    }

    .apply-btn:hover {
        opacity: 0.85;
    }

    .undo-btn {
        background: transparent;
        border: 1px solid rgba(255, 255, 255, 0.15);
        border-radius: var(--radius);
        padding: 0.65rem 1rem;
        color: var(--chalk-dim);
        font-family: var(--font-mono);
        font-size: 0.8rem;
        cursor: pointer;
        transition: all 0.15s;
    }

    .undo-btn:hover {
        background: var(--bg-raised);
        color: var(--chalk);
    }

    .solved-actions {
        display: flex;
        gap: 0.75rem;
    }

    .next-btn {
        background: var(--accent-green);
        color: var(--bg);
        border: none;
        border-radius: var(--radius);
        padding: 0.75rem 1.5rem;
        font-family: var(--font-mono);
        font-weight: 600;
        font-size: 0.9rem;
        cursor: pointer;
        transition: opacity 0.15s;
    }

    .next-btn:hover {
        opacity: 0.85;
    }

    .retry-btn {
        background: transparent;
        border: 1px solid rgba(255, 255, 255, 0.15);
        border-radius: var(--radius);
        padding: 0.75rem 1.25rem;
        color: var(--chalk-dim);
        font-family: var(--font-mono);
        font-size: 0.85rem;
        cursor: pointer;
        transition: all 0.15s;
    }

    .retry-btn:hover {
        background: var(--bg-raised);
        color: var(--chalk);
    }

    .history {
        background: var(--bg-surface);
        border: 1px solid rgba(255, 255, 255, 0.06);
        border-radius: var(--radius);
        padding: 1.25rem;
    }

    .history-title {
        font-family: var(--font-mono);
        font-size: 0.7rem;
        text-transform: uppercase;
        letter-spacing: 0.1em;
        color: var(--chalk-dim);
        margin-bottom: 0.75rem;
    }

    .step-list {
        list-style: none;
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .step {
        display: flex;
        align-items: center;
        justify-content: space-between;
        background: var(--bg);
        border-radius: var(--radius);
        padding: 0.6rem 0.9rem;
        gap: 1rem;
    }

    .step-eq {
        font-family: var(--font-display);
        font-size: 1rem;
        color: var(--chalk-dim);
    }

    .step-meta {
        display: flex;
        align-items: center;
        gap: 0.6rem;
        flex-shrink: 0;
    }

    .step-num {
        font-family: var(--font-mono);
        font-size: 0.65rem;
        color: var(--chalk-dim);
        opacity: 0.5;
    }

    .step-property {
        font-family: var(--font-mono);
        font-size: 0.7rem;
        color: var(--accent-teal);
        background: rgba(94, 196, 176, 0.1);
        border-radius: 4px;
        padding: 0.2rem 0.5rem;
    }

    @media (max-width: 700px) {
        .layout {
            grid-template-columns: 1fr;
            padding: 1rem;
        }

        aside {
            display: none;
        }

        .equation-display {
            font-size: 2rem;
        }
    }
</style>
