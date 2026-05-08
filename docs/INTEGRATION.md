# Derretido — Integración con KitchHero Games Hub

## Descripción

Este repo contiene el juego web Derretido (face puzzle). Al hacer push a `main`, el juego se despliega automáticamente al hub [kitchhero-games](https://github.com/m15-mauricioquezada/kitchhero-games) y queda disponible en:

**`https://games-kitchhero-prod.web.app/derretido/`**

---

## Estructura del repo

```
game-derretido/
├── face-puzzle.html                    ← juego completo (~1.37 MB, single-file)
├── SPEC.md                             ← especificación técnica del juego
└── .github/
    └── workflows/
        └── deploy-to-hub.yml           ← CI: copia al hub y dispara deploy
```

> El archivo fuente se llama `face-puzzle.html` pero se publica como `index.html` en el hub.

---

## Flujo de deploy

```
push a main
    │
    ▼
deploy-to-hub.yml
    │  1. Clona kitchhero-games
    │  2. Copia face-puzzle.html → web/public/derretido/index.html
    │  3. Commit + push al hub
    │
    ▼
kitchhero-games detecta cambio en web/public/**
    │
    ▼
firebase-deploy.yml → firebase deploy --only hosting:prod
    │
    ▼
https://games-kitchhero-prod.web.app/derretido/
```

---

## Secrets requeridos

| Secret | Descripción | Dónde configurar |
|--------|-------------|------------------|
| `HUB_DEPLOY_TOKEN` | GitHub PAT con scopes `repo` + `workflow` | Settings → Secrets → Actions |

```bash
# Configurar desde CLI
echo "TOKEN" | gh secret set HUB_DEPLOY_TOKEN --repo m15-mauricioquezada/game-derretido
```

---

## Rebuild del juego

Si se modifica la lógica del juego y se necesita reensamblar `face-puzzle.html` desde los fuentes:

```bash
cd /tmp/face-puzzle-build
python3 build.py
# escribe el nuevo face-puzzle.html en este repo
```

Ver `SPEC.md` sección 4 (Arquitectura del archivo) y sección 3 (Stack técnico) para detalles del build.

---

## Deploy manual de emergencia

Si el CI falla, se puede copiar manualmente:

```bash
git clone https://github.com/m15-mauricioquezada/kitchhero-games.git
cp face-puzzle.html kitchhero-games/web/public/derretido/index.html
cd kitchhero-games
git add web/public/derretido/
git commit -m "deploy(derretido): manual update"
git push
```

El push al hub dispara el deploy de Firebase automáticamente.
