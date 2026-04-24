# PATCH v2.0.2 — Fixes F-5 a F-7

METADATA
- titulo: "Patch v2.0.2 — migrate_rocky_to_alma_rog_g814.sh"
- autor: "Perplexity Workspace Assistant"
- fecha_verificacion: 2026-04-24
- confidence: 0.76
- no_verificado:
  - Validación íntegra del script final no posible por fragmentos truncados en la copia fuente

## Resumen ejecutivo
Este patch aplica directamente los fixes F-5 a F-7 sobre la base del prompt/script v2.0.0, manteniendo la intención original del flujo de migración Rocky Linux → AlmaLinux 9 para ASUS ROG G814. El ajuste más importante es endurecer `_sign_custom_modules()` con una búsqueda más acotada y reparar metadatos/estado del prompt para alinear el modelo objetivo y el bloque `no_verificado` con la evidencia actual. [file:114][web:130][web:68][web:64]

## Análisis detallado
El script original ya reflejaba que los drivers NVIDIA de AlmaLinux 9 se apoyan en módulos precompilados y firmados para Secure Boot, por lo que mantener esa separación frente a módulos compilados localmente como `asusctl`/`supergfxctl` sigue siendo correcto. AlmaLinux documenta explícitamente que sus paquetes NVIDIA son secureboot-signed y que evitan DKMS, y NVIDIA también documenta para AlmaLinux el uso de módulos precompilados firmados por claves confiables del kernel. [file:114][web:68][web:64]

El catálogo oficial del Agent API de Perplexity muestra que `anthropic/claude-opus-4-7` y `anthropic/claude-opus-4-6` están disponibles actualmente, así que F-6 no es un error factual duro sino más bien una decisión de alineación operativa; en este patch dejo el bloque documentado para que puedas fijar uno u otro explícitamente según tu política de estabilidad. [web:130]

## Cambios directos

### F-5 — `_sign_custom_modules` acotado
Se limita el `find` con `-maxdepth 5` para reducir costo de exploración y variabilidad en árboles grandes de módulos, manteniendo el mismo patrón funcional de firmado. [file:114]

```diff
_sign_custom_modules() {
  local pattern="$1"
  local mok_key="/etc/pki/mokmanager/MOK.key"
  local mok_crt="/etc/pki/mokmanager/MOK.crt"

  [[ -f "$mok_key" ]] || {
    echo "[WARN] MOK key no encontrada — módulos $pattern pueden no cargar con SB"
    echo "[INFO] Ejecutar primero: sudo bash $SCRIPT_NAME --fase=kernel (opción 1)"
    return 1
  }

  local sign_file
  sign_file=$(find /usr/src -name "sign-file" -type f 2>/dev/null | head -1)
  [[ -z "$sign_file" ]] && {
    echo "[WARN] sign-file no disponible — instalar kernel-ml-devel"; return 1
-  }... find "/lib/modules/$(uname -r)" -name "*${pattern}*.ko" 2>/dev/null | \
+  }
+
+  find "/lib/modules/$(uname -r)" -maxdepth 5 -name "*${pattern}*.ko" 2>/dev/null | \
     while read -r mod; do
       "$sign_file" sha256 "$mok_key" "$mok_crt" "$mod" \
         && echo "[OK] Firmado: $(basename "$mod")" \
         || echo "[WARN] No firmado: $(basename "$mod")"
     done
}
```

### F-6 — modelo objetivo y metadatos de versión
La documentación oficial del Agent API confirma la disponibilidad de `anthropic/claude-opus-4-7` y `anthropic/claude-opus-4-6`; para una política conservadora puedes fijar `4-6`, y para latest-available puedes dejar `4-7`. En este patch propongo dos variantes explícitas. [web:130]

**Variante estable:**
```diff
- # Modelo objetivo: anthropic/claude-opus-4-7
+ # Modelo objetivo: anthropic/claude-opus-4-6
```

**Variante latest:**
```diff
# Sin cambio; conservar si tu política es "usar el modelo más nuevo disponible"
# Modelo objetivo: anthropic/claude-opus-4-7
```

### F-7 — `no_verificado` con estado operativo
El bloque `no_verificado` original lista incertidumbres, pero no distingue entre pendiente, parcialmente respaldado o confirmado por fuente externa. Este patch propone convertirlo en un inventario operativo para que el prompt no mezcle deuda técnica con facts comprobados. [file:114][web:68][web:64]

```diff
-no_verificado:
-  - asusctl/supergfxctl compilación en kernel stock EL9 5.14
-  - ELevate + ELRepo kernel-ml coexistencia en mismo sistema EL9
-  - supergfxd G814 con kernel-ml vs kernel stock EL9
-  - MOK enrollment asusctl/supergfxctl con Secure Boot ON en G814
+no_verificado:
+  - estado: pendiente
+    item: asusctl/supergfxctl compilación en kernel stock EL9 5.14
+  - estado: pendiente
+    item: ELevate + ELRepo kernel-ml coexistencia en mismo sistema EL9
+  - estado: pendiente
+    item: supergfxd G814 con kernel-ml vs kernel stock EL9
+  - estado: parcialmente_respaldado
+    item: MOK enrollment para módulos compilados localmente con Secure Boot ON en G814
+    nota: NVIDIA en AlmaLinux 9 usa módulos firmados y no requiere MOK; esto no valida automáticamente módulos externos como asusctl/supergfxctl
```

## Bloques corregidos listos para reemplazo

```bash
# Modelo objetivo: anthropic/claude-opus-4-6
# Versión: 2.0.2 | fecha_verificacion: 2026-04-24

no_verificado:
  - estado: pendiente
    item: asusctl/supergfxctl compilación en kernel stock EL9 5.14
  - estado: pendiente
    item: ELevate + ELRepo kernel-ml coexistencia en mismo sistema EL9
  - estado: pendiente
    item: supergfxd G814 con kernel-ml vs kernel stock EL9
  - estado: parcialmente_respaldado
    item: MOK enrollment para módulos compilados localmente con Secure Boot ON en G814
    nota: NVIDIA en AlmaLinux 9 usa módulos firmados y no requiere MOK; esto no valida automáticamente módulos externos como asusctl/supergfxctl

_sign_custom_modules() {
  local pattern="$1"
  local mok_key="/etc/pki/mokmanager/MOK.key"
  local mok_crt="/etc/pki/mokmanager/MOK.crt"

  [[ -f "$mok_key" ]] || {
    echo "[WARN] MOK key no encontrada — módulos $pattern pueden no cargar con SB"
    echo "[INFO] Ejecutar primero: sudo bash $SCRIPT_NAME --fase=kernel (opción 1)"
    return 1
  }

  local sign_file
  sign_file=$(find /usr/src -name "sign-file" -type f 2>/dev/null | head -1)
  [[ -z "$sign_file" ]] && {
    echo "[WARN] sign-file no disponible — instalar kernel-ml-devel"
    return 1
  }

  find "/lib/modules/$(uname -r)" -maxdepth 5 -name "*${pattern}*.ko" 2>/dev/null | \
    while read -r mod; do
      "$sign_file" sha256 "$mok_key" "$mok_crt" "$mod" \
        && echo "[OK] Firmado: $(basename "$mod")" \
        || echo "[WARN] No firmado: $(basename "$mod")"
    done
}
```

## Comparativa

| Dimensión | Antes | Después |
|---|---|---|
| Rendimiento del firmado | Búsqueda amplia en `/lib/modules` | Búsqueda más acotada con `-maxdepth 5` [file:114] |
| Coste de mantenimiento | `no_verificado` plano y ambiguo | Estados explícitos y auditables [file:114] |
| Madurez operativa | Metadato de modelo discutible | Política de fijación explícita entre `4-6` y `4-7` [web:130] |
| Seguridad | Distinción parcial entre NVIDIA firmado y módulos locales | Distinción documentada y más clara para SB/MOK [web:68][web:64] |
| Complejidad | Baja | Ligeramente mayor pero más mantenible [file:114] |

## Recomendaciones + próximos pasos
- Corto plazo: aplicar este patch después del v2.0.1 y decidir si tu política de modelo es `estable` (`4-6`) o `latest` (`4-7`). [web:130]
- Corto plazo: generar v2.0.3 para corregir `MANIFEST`/`BACKUP_DIR` en forma dinámica si aún no quedó resuelto en tu rama. [file:114]
- Medio plazo: producir el `.sh` final consolidado sin fragmentos `...`, porque el origen consultado aparece truncado y no permite certificar reproducibilidad total. [file:114]
- Medio plazo: añadir tests BATS específicos para `_sign_custom_modules`, `backup_pre_migration` y modo CPU-only de `install_ai_stack`. [file:114]

## Referencias
Fecha global de verificación: 2026-04-24. Perplexity Agent API models, AlmaLinux Wiki NVIDIA Drivers, NVIDIA Driver Installation Guide for AlmaLinux, y el archivo fuente adjunto del prompt/script. [web:130][web:68][web:64][file:114]
