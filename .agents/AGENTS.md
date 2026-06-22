# Ohmify and CircuitJS Development Guidelines

This document contains rules, operational guidelines, and context for AI agents working on the Ohmify and CircuitJS1 projects. Future agents must adhere to these rules to ensure safe, stable changes and efficient workflow. If you discover a better pattern or run into a persistent issue, **update this document** to help future agents!

## 1. Project Architecture
The workspace consists of two tightly-coupled projects:
* **CircuitJS1** (`/home/pontsho/Workspace/projects/circuitjs1`): The core circuit simulation engine written in Java and compiled to Javascript via GWT.
* **Ohmify** (`/home/pontsho/Workspace/projects/ohmify`): A Next.js 16 (Turbopack) web application that wraps the CircuitJS engine within an `iframe` (specifically in `components/Simulator.jsx`) to provide an interactive educational platform.

## 2. Making Changes to CircuitJS1 (The Engine)
When you modify Java files or core HTML/CSS in `circuitjs1`, you must compile the changes and manually ship them to Ohmify.

**How to Build:**
1. Navigate to the `circuitjs1` directory.
2. Run `./gradlew makeSite`. 
   * *Note: Do NOT rely on `./gradlew build` alone as it only compiles the Java without running the GWT compiler. `makeSite` runs the full GWT compilation and packages the result.*
3. **DO NOT** just copy the `war/` directory. The `war/` directory only contains static assets. The GWT compiled javascript gets output to a hidden build folder and combined with `war/` inside the `site/` folder.

**How to Ship to Ohmify:**
1. Once `makeSite` succeeds, copy the contents of `site/` to Ohmify's `public/circuitAPI/` directory.
   ```bash
   cp -r /home/pontsho/Workspace/projects/circuitjs1/site/* /home/pontsho/Workspace/projects/ohmify/public/circuitAPI/
   ```
2. This ensures that Ohmify's iframe (`src="/circuitAPI/circuitjs.html"`) serves the newly updated engine.

## 3. Modifying Ohmify (The Web App)
Ohmify is built with Next.js and uses TailwindCSS for styling.

**Simulator UI Integration:**
* The custom simulation UI lives in `components/Simulator.jsx`.
* **CRITICAL: React Event Propagation:** When building custom UI overlays (like dropdown menus) over the simulator iframe, do **not** use `document.addEventListener("click", ...)` to detect outside clicks. Because of how React synthetic events interact with native DOM events, `e.stopPropagation()` in React will not prevent native document listeners from firing, causing menus to instantly close. 
* **The Fix:** Always use a full-screen invisible overlay `div` (e.g., `className="absolute inset-0 z-10"`) with an `onClick` handler to close menus safely.
* The simulator iframe is initialized with query parameters like `?hideMenu=true&hideSidebar=true` to suppress the native CircuitJS UI so our custom React UI can take over.

## 4. Running Tests
* **Automated Scripts:** There are Puppeteer test scripts located in the `ohmify` root (e.g., `test_ui_class.js`, `test_ui_screenshot.js`). Use these via `node <script_name>` to quickly validate UI states and class additions without needing a browser window.
* **Dev Server:** Run `npm run dev` to start the Next.js server for manual testing. *Note: Port 3000 might occasionally be locked; Next.js will fallback to 3001, so ensure your Puppeteer scripts point to the right port if testing against the dev server.*

## 5. Iteration and Safety
* **Avoid Silent Failures:** If you edit `Simulator.jsx` and add React state, be exceedingly careful with syntax (e.g., unmatched braces). 
* **State Syncing:** Remember that the React UI and the CircuitJS iframe are two different worlds. Use `sim.triggerMenu(menu, action)` to map React button clicks to native CircuitJS operations.
* **Continuous Improvement:** If you find a new edge case or a better way to structure testing between the two repos, update this `AGENTS.md` file immediately!

## 6. Safely Publishing Updates to Production (scope.io)
Currently, changes to the CircuitJS engine are manually copied over via `cp` to Ohmify before deployment. To safely and reliably publish updates to the main app (`scope.io`), the recommended long-term workflow is to automate this via GitHub to eliminate manual file-copying steps and ensure version control integrity.

**Recommended Automated Workflow (via GitHub Actions):**
1. **GitHub Actions for CircuitJS:** Create a CI/CD workflow in the `circuitjs1` repository that automatically runs `./gradlew makeSite` upon merging into the `main` branch.
2. **Publishing the Compiled Artifact:**
   - **Method A (NPM Package - Highly Recommended):** Have the GitHub Action publish the compiled `site/` output as a private NPM package (e.g., `@ohmify/circuit-engine`). Ohmify will list this as a standard dependency in `package.json`. A custom `postinstall` script in Ohmify will automatically copy the files from `node_modules/@ohmify/circuit-engine/site` into `public/circuitAPI/`. This seamlessly locks engine versions to specific Next.js deployments.
   - **Method B (Git Submodule):** Have the GitHub Action commit the `site/` directory to a separate orphaned `build` branch. Ohmify can then pull this `build` branch in as a Git Submodule directly mapped to `public/circuitAPI/`. You can update the engine by simply running `git submodule update --remote`.
3. **Deploying to scope.io:** Once Ohmify fetches the updated engine via NPM or Git, simply push Ohmify to its `main` branch to trigger your standard deployment (e.g. Vercel) to `scope.io`. No manual `cp` required, and the entire team will always be on the correct engine version.
