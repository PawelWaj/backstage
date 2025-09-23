```markdown
# Backstage Plugin Bootcamp – Complete Markdown Steps  
---

## Rung 0 – One-time setup (15 min)

1. Install Node 18 LTS + Yarn 1.22  
```bash
nvm install 18 && nvm use 18
npm i -g yarn
```

2. Create the monorepo once and **never delete it** – you’ll reuse it for every experiment.  
```bash
npx @backstage/create-app@latest   # name: my-backstage
cd my-backstage
```

3. Start the stack  
```bash
yarn dev
```  
→ http://localhost:3000 (frontend) + http://localhost:7007 (backend)  
✅ Checkpoint: you see the vanilla Backstage home page.

---

## Rung 1 – Hello-World plugin (15 min)

1. Scaffold a frontend plugin  
```bash
yarn new          # pick “frontend-plugin”, id: hello
```

2. Start the plugin dev server  
```bash
yarn workspace @backstage/plugin-hello start
```

3. Browser → http://localhost:3000/hello  
✅ You see the generated template page.

---

## Rung 2 – Read **public** API (30 min)

Goal: display live data from an open REST endpoint (no auth).

1. Install helper  
```bash
yarn add react-use
```

2. Replace `plugins/hello/src/components/ExampleComponent.tsx` with:

```tsx
import React from 'react';
import { useAsync } from 'react-use';
import { Table, TableColumn, Progress } from '@backstage/core-components';

type CatFact = { fact: string; length: number };

export const ExampleComponent = () => {
  const { value, loading, error } = useAsync(async (): Promise<CatFact[]> => {
    const r = await fetch('https://catfact.ninja/facts?limit=5');
    return (await r.json()).data;
  }, []);

  if (loading) return <Progress />;
  if (error) return <div>{error.message}</div>;

  const cols: TableColumn[] = [
    { title: 'Fact', field: 'fact' },
    { title: 'Length', field: 'length' },
  ];
  return <Table title="Cat facts" columns={cols} data={value || []} />;
};
```

3. Save → hot-reload → table with 5 cat facts.  
✅ Lesson: any REST → React table pattern mastered.

---

## Rung 3 – Proxy + backend plugin (45 min)

Goal: hide secrets, avoid CORS, keep traffic server-side.

1. Create backend plugin  
```bash
yarn new   # pick “backend-plugin”, id: catfacts
```

2. Add proxy config in `app-config.yaml`:

```yaml
proxy:
  '/catfacts':
    target: https://catfact.ninja
    changeOrigin: true
    pathRewrite:
      '/api/proxy/catfacts': ''
```

3. Backend router (`plugins/catfacts/src/service/router.ts`):

```ts
import { Router } from 'express';
import fetch from 'node-fetch';

export async function createRouter(): Promise<Router> {
  const router = Router();
  router.get('/facts', async (_req, res) => {
    const r = await fetch('https://catfact.ninja/facts?limit=5');
    res.json(await r.json());
  });
  return router;
}
```

4. Wire router into `packages/backend/src/index.ts` (CLI prints the exact lines).

5. Frontend swap:

```tsx
import { discoveryApiRef, useApi } from '@backstage/core-plugin-api';

const base = await useApi(discoveryApiRef).getBaseUrl('catfacts');
const r = await fetch(`${base}/facts`);
```

✅ Same table, but traffic flows through backend → ready for auth.

---

## Rung 4 – Add **real** third-party with auth (60 min)

Use GitHub REST API (private repo commits).

1. GitHub PAT  
```bash
export GITHUB_TOKEN=ghp_xxxx
```

2. `app-config.local.yaml` (git-ignored):

```yaml
proxy:
  '/github':
    target: https://api.github.com
    headers:
      Authorization: Bearer ${GITHUB_TOKEN}
      X-GitHub-Api-Version: '2022-11-28'
```

3. Frontend fetch: `/api/proxy/github/repos/ORG/REPO/commits`

4. Catalog-aware: read repo slug from entity annotation:

```tsx
import { useEntity } from '@backstage/plugin-catalog-react';
const { entity } = useEntity();
const slug = entity.metadata.annotations?.['github.com/project-slug'] || 'kubernetes/kubernetes';
```

✅ Commits table shows per-component repo data.

---

## Rung 5 – Turn it into a **Catalog tab** (30 min)

Goal: “Commits” tab inside Component page.

1. Export routable extension in `plugins/hello/src/plugin.ts`:

```ts
export const CommitsContent = myPlugin.provide(
  createComponentExtension({
    name: 'CommitsContent',
    component: { lazy: () => import('./components/CommitsComponent').then(m => m.CommitsComponent) },
  }),
);
```

2. Add to `packages/app/src/components/catalog/EntityPage.tsx`:

```tsx
import { CommitsContent } from '@internal/plugin-hello';

<EntityLayout>
  ...
  <EntityLayout.Route path="/commits" title="Commits">
    <CommitsContent />
  </EntityLayout.Route>
</EntityLayout>
```

✅ Tab appears for every Component → fully integrated.



---

## Micro-roadmap

Day 1 → Rungs 0-2  
Day 2 → Rungs 3-4  
Day 3 → Rungs 5-6  


