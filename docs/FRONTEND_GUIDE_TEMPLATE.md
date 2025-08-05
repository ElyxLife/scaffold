# Universal Frontend Development Guide Template

## How to use this document
1. Copy or rename to `docs/FRONTEND_GUIDE.md` in your repo.
2. Replace placeholder tokens (e.g. `<<APP_NAME>>`).
3. Remove sections that don’t apply.

---

## 1  Development Standards
- Develop **exclusively in TypeScript**—no plain JavaScript files.
- Use **shadcn/ui** components to ensure visual consistency.
- Style via **Tailwind CSS** utility classes; avoid ad-hoc CSS files.
- Configure the path alias **`@/*` → `src/`** for cleaner imports.
- Organise files by responsibility: `pages/`, `components/`, `services/`, `hooks/`.

## 2  Testing Guidelines
- Write tests with **Vitest** + **React Testing Library**—focus on user behaviour, not implementation details.
- Mock network requests using **axios-mock-adapter** (or equivalent) to keep tests deterministic.
- Target the same **≥ 80 % coverage** threshold enforced in backend.

## 3  Recommended CI Checks
- Lint with **ESLint** and enforce formatting with **Prettier**.
- Run `npm run build` (or `next build`, etc.) in CI to catch compile-time errors.
