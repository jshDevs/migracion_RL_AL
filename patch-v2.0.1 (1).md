# PATCH v2.0.1 — Fixes F-1 a F-4
# fecha_verificacion: 2026-04-24
# objetivo: corregir defectos críticos/altos detectados en v2.0.0

## 1) Header / metadatos

```diff
- # Modelo objetivo: anthropic/claude-opus-4-7
+ # Modelo objetivo: anthropic/claude-opus-4-6

- # Versión: 2.0.0 | fecha_verificacion: 2026-04-24
+ # Versión: 2.0.1 | fecha_verificacion: 2026-04-24

- # migrate_rocky_to_alma_rog_g814.sh v2.0.0
+ # migrate_rocky_to_alma_rog_g814.sh v2.0.1

- readonly SCRIPT_VERSION="2.0.0"
+ readonly SCRIPT_VERSION="2.0.1"
```

## 2) Fix F-4 — `BACKUP_DIR` no debe ser `readonly`

```diff
- readonly BACKUP_DIR="${BACKUP_ROOT}/pre-migration-$(date +%Y%m%d-%H%M%S)"
+ BACKUP_DIR="${BACKUP_ROOT}/pre-migration-$(date +%Y%m%d-%H%M%S)"
```

Motivo: `backup_pre_migration()` reutiliza un backup existente y reasigna `BACKUP_DIR`; con `readonly` eso aborta la ejecución.

## 3) Fix F-1 — `detect_hardware()` sintaxis rota

```diff
 detect_hardware() {
   local dmi
   dmi=$(cat /sys/class/dmi/id/product_name 2>/dev/null || echo "unknown")
   echo "$dmi" | grep -qi "G814\|ROG" && IS_ASUS_ROG_G814=1
-  echo "[INFO] Hardware: $dmi (ROG G814 detectado: ${IS_ASUS_ROG_G814})"... lspci | grep -qi "NVIDIA" && HAS_NVIDIA_GPU=1
+  echo "[INFO] Hardware: $dmi (ROG G814 detectado: ${IS_ASUS_ROG_G814})"
+  lspci | grep -qi "NVIDIA" && HAS_NVIDIA_GPU=1
   [[ $HAS_NVIDIA_GPU -eq 1 ]] &&      echo "[INFO] GPU NVIDIA: $(lspci | grep -i NVIDIA | head -1)"
```

## 4) Fix F-3 — `RESOLVED_PYTORCH_WHL` truncado

```diff
   RESOLVED_CUDA_PKG="cuda-toolkit-${cuda_pkg_ver}"
   RESOLVED_CUDNN_PKG="libcudnn9-cuda-${c_maj}"
-  RESOLVED_PYTORCH_WHL="https://download.pytorch.org/whl/cu${c_maj}${c_min}"...
+  RESOLVED_PYTORCH_WHL="https://download.pytorch.org/whl/cu${c_maj}${c_min}"
```

## 5) Fix F-2 — `install_ai_stack()` cierre y flujo CPU-only

```diff
 install_ai_stack() {
   [[ $SKIP_AI -eq 1 ]] && { echo "[SKIP] Stack AI omitido (--skip-ai)"; return 0; }
   [[ $HAS_NVIDIA_GPU -eq 0 ]] && {
-    echo "[WARN] Sin GPU NVIDIA — AI/ML en CPU únicamente"; 
+    echo "[WARN] Sin GPU NVIDIA — AI/ML en CPU únicamente"
   }
   [[ $DRY_RUN -eq 1 ]] && { echo "[DRY-RUN] AI stack omitido"; return 0; }

-  configure_gpu_for_ai
+  [[ $HAS_NVIDIA_GPU -eq 1 ]] && configure_gpu_for_ai
   install_miniconda
   setup_ai_environments
   install_ollama
-  install_llamacpp_cuda
+  [[ $HAS_NVIDIA_GPU -eq 1 ]] && install_llamacpp_cuda
   install_jupyterlab
 }
```

## 6) Bloque corregido listo para reemplazo

```bash
#!/usr/bin/env bash
# migrate_rocky_to_alma_rog_g814.sh v2.0.1
# Rocky Linux → AlmaLinux 9 | ASUS ROG G814/G18 2023 + AI/ML Stack
# Uso: sudo bash migrate_rocky_to_alma_rog_g814.sh [--fase=FASE] [opciones]
# Opciones:
#   --fase=FASE          : preflight|backup|migrate|kernel|compat|nvidia|
#                          asus|wifi|ai|harden|qa
#   --battery-limit=N    : límite de carga 20-100 (default: 80)
#   --skip-ai            : omitir instalación de stack AI/ML
#   --skip-asus          : omitir compilación asusctl/supergfxctl
#   --dry-run            : mostrar plan sin ejecutar cambios

set -euo pipefail
IFS=$'
	'

readonly SCRIPT_VERSION="2.0.1"
readonly SCRIPT_NAME="$(basename "$0")"
readonly MIGRATION_LOG="/var/log/migration-$(date +%Y%m%d-%H%M%S).log"
readonly BACKUP_ROOT="/var/backup/migration"
BACKUP_DIR="${BACKUP_ROOT}/pre-migration-$(date +%Y%m%d-%H%M%S)"
readonly MANIFEST="${BACKUP_DIR}/manifest.sha256"
readonly BUILD_BASE="/tmp/asus-build-${BASHPID}"
readonly LOCK_FILE="/var/run/migration-rog-g814.lock"

detect_hardware() {
  local dmi
  dmi=$(cat /sys/class/dmi/id/product_name 2>/dev/null || echo "unknown")
  echo "$dmi" | grep -qi "G814\|ROG" && IS_ASUS_ROG_G814=1
  echo "[INFO] Hardware: $dmi (ROG G814 detectado: ${IS_ASUS_ROG_G814})"
  lspci | grep -qi "NVIDIA" && HAS_NVIDIA_GPU=1
  [[ $HAS_NVIDIA_GPU -eq 1 ]] &&     echo "[INFO] GPU NVIDIA: $(lspci | grep -i NVIDIA | head -1)"
}

install_ai_stack() {
  [[ $SKIP_AI -eq 1 ]] && { echo "[SKIP] Stack AI omitido (--skip-ai)"; return 0; }
  [[ $HAS_NVIDIA_GPU -eq 0 ]] && {
    echo "[WARN] Sin GPU NVIDIA — AI/ML en CPU únicamente"
  }
  [[ $DRY_RUN -eq 1 ]] && { echo "[DRY-RUN] AI stack omitido"; return 0; }

  [[ $HAS_NVIDIA_GPU -eq 1 ]] && configure_gpu_for_ai
  install_miniconda
  setup_ai_environments
  install_ollama
  [[ $HAS_NVIDIA_GPU -eq 1 ]] && install_llamacpp_cuda
  install_jupyterlab
}

# dentro de detect_and_resolve_nvidia_cuda_compatibility()
# reemplazar la línea de wheel por:
# RESOLVED_PYTORCH_WHL="https://download.pytorch.org/whl/cu${c_maj}${c_min}"
```

## 7) Riesgo residual

- `MANIFEST` sigue siendo `readonly` y depende del valor inicial de `BACKUP_DIR`; si reutilizas un backup existente, el manifest efectivo puede desalinearse del backup reutilizado. Conviene corregirlo en v2.0.2 con una variable calculada por función o convirtiendo `MANIFEST` en dinámico.
- El archivo fuente parece truncado en varios puntos con `...`, así que este patch corrige F-1 a F-4 pero no certifica integridad completa del script. Se requiere revisión íntegra antes de marcar `confidence ≥ 0.85`.
