# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

Single-file PowerShell maintenance script for Windows 10/11, published at `https://github.com/audrum/mantenimiento-windows`. All text (comments, messages, README) is in **Spanish**.

## Critical encoding requirement

After every edit to `Mantenimiento-Windows.ps1`, run this Python one-liner to re-apply **UTF-8 with BOM** and **CRLF** line endings. Windows PowerShell 5.1 will throw cryptic parser errors (e.g. `Missing expression after unary operator '--'`) if either is missing:

```bash
python3 -c "
with open('Mantenimiento-Windows.ps1', 'r', encoding='utf-8-sig') as f:
    content = f.read()
crlf = '\r\n'.join(content.splitlines()) + '\r\n'
with open('Mantenimiento-Windows.ps1', 'wb') as f:
    f.write(b'\xef\xbb\xbf')
    f.write(crlf.encode('utf-8'))
print('BOM + CRLF aplicado correctamente')
"
```

## Architecture

### run.ps1 (bootstrapper)

`run.ps1` is a minimal launcher that enables the `irm ... | iex` one-liner workflow:
- Accepts the same 4 parameters as the main script (`-AutoReiniciar`, `-SegundosEspera`, `-Pasos`, `-TodosLosPasos`).
- If the session is **not elevated**, self-elevates via `Start-Process powershell -Verb RunAs`, re-downloading itself with the same parameters forwarded.
- If already elevated, downloads `Mantenimiento-Windows.ps1` via `Invoke-RestMethod` and runs it as a scriptblock with splatted parameters.
- Saved as **UTF-8 with BOM + CRLF** (same as the main script).

### Mantenimiento-Windows.ps1 (main script)

`Mantenimiento-Windows.ps1` is a single flat file structured in clearly delimited sections (`# ===... SECCION ...===`):

1. **Parameters** — `AutoReiniciar`, `SegundosEspera`, `Pasos`, `TodosLosPasos`
2. **Global config** — `$Script:Version`, log directory/file path, `$Script:Resumen` list
3. **Step array** — `$Script:PasosDisponibles`: a plain `@()` array of hashtables, each with keys `Numero`, `Nombre`, `Desc`, `Funcion` (scriptblock), `Requerido`, `Seleccionado`
4. **Utility functions** — `Escribir-Log`, `Agregar-Resumen`, `Obtener-TamanoLegible`, `Limpiar-DirectorioSeguro`
5. **Step functions** (one per step, named descriptively in Spanish)
6. **Helper functions** — `Calcular-PorcentajeSalud` (health % for step 15), `Mostrar-Resumen`
7. **Menu function** — `Mostrar-MenuPasos` (interactive checkbox UI)
8. **Restart function** — `Solicitar-Reinicio`
9. **Main execution block** — resolves step selection, loops over `$Script:PasosDisponibles`, calls each `$paso.Funcion`

### Key conventions

- **`Escribir-Log`** writes to both console (colored by `-Tipo`: `OK`=green, `WARN`=yellow, `ERROR`=red, `SECCION`=cyan, `INFO`=white) and the `.log` file simultaneously. Always use it instead of bare `Write-Host` + `Add-Content`, except when a custom `ForegroundColor` is needed (e.g. the health bar in step 15).
- **Step array** must remain a plain `@()` array of hashtables — never switch back to `[ordered]@{}`. PowerShell's `OrderedDictionary` coerces string keys to int positional indexers, which caused out-of-range bugs with 14+ entries.
- **Step 1 is always required** (`Requerido=$true`) and cannot be deselected in the menu.
- **SSD vs HDD detection** is done per-disk via `$disco.MediaType -eq "HDD"` inside each step function; the global `$Script:EsSSD` flag set in step 1 (`Obtener-InfoSistema`) is available for other functions.

## Adding a new step

1. Add a hashtable entry to `$Script:PasosDisponibles` (increment `Numero`, provide a scriptblock in `Funcion`).
2. Write the step function above the `Mostrar-MenuPasos` function.
3. Bump `$Script:Version`.
4. Re-apply BOM + CRLF (see above).
5. Update README.md: step table row, dedicated section if needed, version history entry.

## Commit and push workflow

```bash
git add Mantenimiento-Windows.ps1 run.ps1 README.md
git commit -m "mensaje descriptivo"
git push origin master
```

Remote is SSH (`git@github.com:audrum/mantenimiento-windows.git`).
