# Slop check

Searches to run on the code you changed before calling it done. Hits are leads, not verdicts: fix each one or say why it's fine (e.g. `animate-pulse` on a skeleton is fine).

The commands use ripgrep (`rg`) in bash or zsh, run from the project root. If `rg` isn't available, use your own search tool with the same patterns. First define `s`, which searches source files only. Always give `rg` a path (the trailing `.`): without one, in a non-interactive shell, it waits on stdin and hangs.

```sh
s() { rg -n -g '*.{tsx,jsx,ts,js,mjs,vue,svelte,astro,html,css,scss,erb,heex}' "$@" .; }
```

## AI-look tells

```sh
s 'bg-(indigo|violet|purple)-[0-9]|from-(indigo|violet|purple)-[0-9]|#6366f1|#8b5cf6|#7c3aed|#a855f7'   # stock AI accent
s 'bg-clip-text'                                  # gradient text
s 'transition-all|transition:\s*all'              # animating everything
s 'hover:scale-|hover:-translate-y'               # bouncy cards
s 'animate-(bounce|pulse|ping)'                   # motion spam (fine on skeletons)
s 'border-l-(4|8)\b|border-left:\s*[3-9]px'       # side-stripe cards
s 'uppercase[^"]*tracking-(wide|wider|widest)'    # eyebrow labels
s '\p{Extended_Pictographic}'                     # emoji in UI
s -c 'shadow-(xl|2xl)|rounded-(2xl|3xl)'          # card reflex: high counts are a smell
s -c '\[[0-9]+px\]|#[0-9a-fA-F]{6}\b'             # one-off values outside the theme
s '\w (·|•|&middot;|&bull;) \w'                   # middle dots between words
s '>\s*0[1-9]\s*[<./]'                            # numbered sections (01, 02)
s '<h[1-3][^>]*>[^<]*<(em|i)\b'                   # one italic word in a headline
s -i 'instrument[ _+-]?serif|dither'              # cliché font (fine if DESIGN.md chose it), dithering
s 'blur-(2xl|3xl)|blur\(([4-9][0-9]|[1-9][0-9]{2})px\)'   # glow blobs
```

## Copy tells

```sh
s -i '\b(unlock|unleash|empower|supercharge|elevate|revolutioni[sz]e|seamless(ly)?|cutting-edge|next-gen(eration)?|world-class|game-chang|delve|leverage)\b'
s -i 'lorem ipsum|john doe|jane doe|coming soon|placeholder text'
s -i '>[^<{]*\b(quiet(ly)?|real)\b'               # "quiet" and "real" in visible text; check copy files too
s -i 'oops|something went wrong|are you sure\?|invalid (input|email|value)|this field is required'
s '\b(window\.)?(alert|confirm|prompt)\('
s '>\s*(Submit|OK|Click here)\s*<'
```

## Security holes

```sh
s 'dangerouslySetInnerHTML|v-html|\.innerHTML\s*=|\beval\(|new Function\('
s -i 'localStorage\.setItem\([^)]*(token|jwt|session|auth)'
s '\$queryRawUnsafe|\$executeRawUnsafe|Prisma\.raw\('
s '(query|execute|raw)\(\s*`[^`]*\$\{'            # SQL built with template strings
s 'child_process|\bexec\(\s*`'                     # shell commands
s 'data:\s*(req\.body|body|await req\.json\(\))'  # mass assignment
s 'cors\(\s*\{[^}]*origin:\s*(true|.\*.)'          # open CORS
s 'jwt\.decode\(|rejectUnauthorized:\s*false'
s "postMessage\(.*['\"]\*['\"]"                    # postMessage to any origin
s '(json|send)\([^)]*(err|error)\.(stack|message)' # leaking errors to clients
rg -n -i 'using\s*\(\s*true\s*\)|if\s+true\s*;' -g '*.{sql,rules}' .   # open RLS / Firebase rules
```

Secrets behind public prefixes, including in `.env` files (hidden and usually gitignored, so search them explicitly):

```sh
rg -n --hidden --no-ignore -g '!node_modules' -g '!.git' -i '(NEXT_PUBLIC|VITE|EXPO_PUBLIC|REACT_APP|PUBLIC)_[A-Z0-9_]*(SECRET|SERVICE_ROLE|PRIVATE|OPENAI|ANTHROPIC|STRIPE_SECRET|DATABASE)' .
```

Supabase: every `create table` in your migrations needs a matching `enable row level security`:

```sh
rg -c -i 'create table' supabase/migrations; rg -c -i 'enable row level security' supabase/migrations
```

## After a build

Secrets in the shipped bundle (use whichever folders exist: `.next/static`, `dist`, `build`, `out`):

```sh
rg -n --no-ignore 'sk_live_|sk-[A-Za-z0-9_-]{20,}|sk-ant-|service_role|whsec_|postgres(ql)?://|mongodb\+srv://|BEGIN (RSA |EC )?PRIVATE KEY' .next/static dist build out
```

Headers on the deployed site:

```sh
curl -sI https://your-app.example | rg -i 'content-security-policy|strict-transport|x-content-type|referrer-policy|x-frame-options|x-powered-by'
```

Dependencies:

```sh
npm audit --audit-level=high     # or pnpm audit / yarn npm audit
```
