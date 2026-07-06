# Manual de Referencia SynxFlow HPC

Manual de referencia para la simulación hidrodinámica de amenazas naturales en HPC con SynxFlow (español / English), publicado con MkDocs Material.

Elaborado por Carlos Millán-Arancibia (SENAMHI, Perú) como parte del desarrollo del Sistema de Pronóstico de Activación de Quebradas para la Gestión del Riesgo en la cuenca media-baja del río Rímac (ARGO-Rímac).

## Desarrollo local

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Abrir http://127.0.0.1:8000/.

## Estructura

```
docs/
  es/   # contenido en español (idioma por defecto)
  en/   # contenido en inglés
```

## Publicación

El sitio se despliega automáticamente a GitHub Pages mediante GitHub Actions (`.github/workflows/deploy.yml`) en cada push a `main`. Requiere habilitar Pages en la configuración del repo, apuntando a la rama `gh-pages`.
