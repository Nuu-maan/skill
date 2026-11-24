# UX writing

AI copy is easy to spot: hype verbs, three-beat slogans, vague claims, cute errors. Real products name the thing and the action.

## Rules

1. **Buttons**: verb + object that names the result. "Save API key", "Delete project", "Send invite". Not "OK", "Submit" or "Let's go!". Dialog buttons describe each outcome: "Delete file" / "Keep file".
2. **One term per concept** everywhere: buttons, headings, toasts, emails. Pick "project" or "workspace", not both. List the terms in DESIGN.md.
3. **Sentence case** for headings, buttons and labels. No periods on headings or labels.
4. **Field errors** say what to do, in the label's words: "Enter your email address", "Name must be 35 characters or fewer", "Enter an email like sam@acme.com". Never "Invalid", "This field is required", "Oops", "please" or "you forgot".
5. **System errors**: what failed + what to do. "Couldn't save changes. Check your connection and try again." Apologize only for real outages. No jokes in errors.
6. **Confirmations** name the object and the consequence: "Delete 'Q3 Report'? This can't be undone." Never "Are you sure?".
7. **Success** is short and factual: "Invite sent", "Changes saved". Not "Awesome! Your changes have been saved successfully!"
8. **Progress** uses a real ellipsis: "Saving…", "Uploading 3 files…". Menu items that open a dialog end in "…".
9. **Empty states**: what goes here + why it's useful + the action. "No invoices yet. Invoices you send appear here. [Create invoice]". Filtered-empty is different: "No invoices match these filters. [Clear filters]".
10. **Link text** names the destination: "View billing settings", not "click here" or a bare "Learn more".
11. **Cut filler**: "Dashboard", not "Your Dashboard". "Total", not "This is the total". Drop "Please note", "Simply", "Just".
12. **Plain words**: "buy" not "purchase", "use" not "utilize", "help" not "assist". Keep sentences under about 25 words.
13. **Multi-step flows** use one word for moving forward ("Continue" or "Next") throughout, and "Done" at the end.
14. **Ask like a person**: "How would you like to pay?", not "Please select your preferred payment method."
15. **Settings** describe what happens when on: "Email me when someone comments".
16. **Specifics beat adjectives**: "Syncs in under a second" beats "Blazing fast".
17. **Landing-page claims** pass three tests: can you picture it, could it be proven false, could only this product say it?

## AI-copy tells → what to write

| Tell | Write instead |
|---|---|
| Unlock, Unleash, Empower, Supercharge, Elevate, Revolutionize | The mechanism + outcome: "Send invoices and get paid by card" |
| "The AI-powered X for modern Y" | The outcome for a named buyer: "Payroll for restaurants with hourly staff" |
| "Save time. Work smarter. Scale faster." | One specific claim, ideally with a number |
| "It's not just X, it's Y" / "No X. No Y. Just Z." | Just say Y |
| Seamless, robust, cutting-edge, world-class, next-generation | A fact someone could check |
| delve, leverage, foster, tapestry, pivotal, crucial, showcase | Plain verbs: use, build, show, important |
| Em dashes in every sentence | A period, comma or colon |
| Emoji at the start of headings or bullets | Nothing |
| Title Case On Every Heading | Sentence case |
| "Oops! Something went wrong" | "Couldn't load invoices. [Retry]" |
| "Welcome back, John! 👋" | The user's actual next task, or nothing |
| "10,000+ happy customers", placeholder logos, invented testimonials | Real numbers or nothing |
| "Get started" on every button | The specific action: "Create workspace" |

Listing banned words can prime a model to use them. Always write the replacement, not just the avoidance.
