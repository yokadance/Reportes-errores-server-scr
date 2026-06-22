# REPORTE DE INCIDENTE - VM YaeServer
## XCP-ng Host: xcp10 (192.168.1.12)

---

## RESUMEN

- **Fecha del Incidente:** 19 de Junio de 2026
- **Fecha de Resolución:** 22 de Junio de 2026
- **Duración Total:** ~3 días
- **Sistema Afectado:** VM YaeServer (UUID: 1fda53a2-a31b-61ca-e419-0866cb44b134)
- **Causa Raíz:** Fallo del subsistema tapdisk a nivel de kernel/hypervisor
- **Solución Aplicada:** Reinicio completo del hypervisor XCP-ng

---

## DETALLES DEL INCIDENTE

### Síntomas Iniciales (19 de Junio 2026)

La VM "YaeServer" no podía iniciar, mostrando el siguiente error:

```
Error code: SR_BACKEND_FAILURE_453
Error parameters: , tapdisk experienced an error [opterr=Operation not permitted],
```

### Análisis Técnico Realizado

#### 1. Verificación de VBDs (Virtual Block Devices)
- **Problema detectado:** VBD de CD-ROM con referencia rota `<not in database>`
  - UUID del VBD problemático: `ad97ae89-f38d-0953-5381-626eb3fe7bba`
  - Acción: Se eliminó el VBD corrupto con `xe vbd-destroy`

- **Disco principal:**
  - VDI UUID: `46cca168-69a4-4cd5-86fd-2f9ff35f10c3`
  - Nombre: CentOS 7_ilafa
  - Tamaño: 100.20 GB
  - SR: Local storage (UUID: 58956d9f-dfae-9ef2-9ef7-87b531468267)
  - Estado: VHD válido (verificado con `vhd-util check`)

#### 2. Verificación de Storage Repository
- Tipo: LVM
- Espacio usado: 595 GB / 1955 GB (30%)
- Estado: Operativo
- LVM: Logical volume activo y accesible

#### 3. Pruebas de Tapdisk

**Hallazgos críticos:**
```bash
# Test 1: Crear tapdisk para el VDI específico
tap-ctl create -a vhd:/dev/VG_XenStorage-.../VHD-46cca168...
Resultado: Operation not permitted ❌

# Test 2: Spawn de tapdisk genérico
tap-ctl spawn
Resultado: Operation not permitted ❌

# Test 3: Lectura directa del dispositivo
dd if=/dev/mapper/VG_XenStorage-.../VHD-46cca168... of=/dev/null bs=512 count=1
Resultado: SUCCESS ✓
```

**Conclusión:** El subsistema tapdisk a nivel de kernel estaba bloqueado para crear nuevos procesos, aunque los tapdisks existentes (VMs corriendo) funcionaban normalmente.

#### 4. Acciones de Troubleshooting Realizadas

| Acción | Resultado | Observaciones |
|--------|-----------|---------------|
| Restart toolstack (`xe-toolstack-restart`) | ❌ Fallo | No resolvió el problema |
| SR scan | ✓ Completado | Sin efecto en el error |
| Eliminación VBD corrupto | ✓ Completado | No resolvió el arranque |
| Intento de copia VDI (`vdi-copy`) | ❌ Fallo | Mismo error tapdisk |
| Verificación SELinux/AppArmor | N/A | Ambos deshabilitados |
| Verificación blktap kernel module | ✓ Cargado | Módulo builtin funcionando |

#### 5. Recursos del Sistema

**Blktap devices activos (antes del reinicio):**
- 4 tapdisk processes corriendo (para VMs: RRHH, MySQL-Server, Samba, y otra)
- Devices: /dev/xen/blktap-2/blktap0 a blktap5
- Último dispositivo creado: blktap4 (19 Jun 16:59)

**File descriptors:**
- Límite del sistema: 741,952
- En uso: 2,528
- Límite soft: 1,024
- Límite hard: 4,096

---

## RESOLUCIÓN

### Decisión de Reinicio (22 de Junio 2026)

Dado que:
1. Todas las acciones de troubleshooting a nivel de software fallaron
2. El problema era a nivel de kernel (tap-ctl spawn fallaba)
3. No existían alternativas viables sin downtime

Se procedió a:
1. Apagar ordenadamente todas las VMs corriendo (RRHH, MySQL-Server, Samba)
2. Reiniciar el hypervisor XCP-ng

### Proceso de Reinicio

**Timeline:**
- **Primera

 tentativa:** 22 Jun 09:00 - Falló (duración: 2 minutos)
- **Segunda tentativa:** 22 Jun 09:05 - Exitosa
- **VMs operativas:** 22 Jun ~12:10

**VMs afectadas por el reinicio:**
- RRHH
- MySQL-Server
- Samba
- YaeServer (objetivo)

**Downtime aproximado:** 3 horas (09:00 - 12:10)

---

## ESTADO POST-RESOLUCIÓN

### Sistema Actual (22 de Junio 2026 - 10:45 AM)

**Host XCP-ng:**
- Versión: XCP-ng 8.3.0 (build cloud)
- Uptime: 1 hora 40 minutos
- Kernel: 4.19.0+1
- Xen: 4.17.4-6

**Todas las VMs:**
```
✓ RRHH               - running
✓ MySQL-Server       - running
✓ Samba              - running
✓ YaeServer          - running (start time: 22 Jun 12:10:55 UTC)
  RESTORE_VM-113     - halted (normal)
```

**Tapdisk status actual:**
- 5 procesos tapdisk corriendo normalmente
- YaeServer usando minor=4, PID 8307
- Todos los VHDs accesibles sin errores

---

## CAUSA RAÍZ IDENTIFICADA

**Hipótesis más probable:**

Agotamiento de recursos o corrupción en el subsistema blktap/tapdisk del kernel XCP-ng, que impidió la creación de nuevos procesos tapdisk sin afectar los existentes.

**Factores contribuyentes posibles:**
1. Uptime prolongado del hypervisor (14+ días desde último reinicio: 5 Jun)
2. Posible memory leak en el driver blktap
3. Límite de recursos internos del kernel alcanzado
4. VBD corrupto (CD-ROM con referencia inválida) que pudo desencadenar el problema

**Evidencia:**
- `tap-ctl spawn` fallaba con "Operation not permitted"
- Operación de bajo nivel (dd) sobre el VDI funcionaba correctamente
- VMs existentes continuaban funcionando normalmente
- Reinicio del hypervisor resolvió completamente el problema

---

## RECOMENDACIONES

### Inmediatas
1. ✅ **Completado:** Todas las VMs operativas incluyendo YaeServer
2. ⚠️ **Monitorear:** Estado de tapdisk durante las próximas 48 horas

### Corto Plazo (1-2 semanas)
1. **Implementar monitoreo de tapdisk:**
   ```bash
   # Agregar a cron para verificar health de tapdisk
   */15 * * * * tap-ctl list > /var/log/tapdisk-health.log
   ```

2. **Verificar y limpiar VBDs huérfanos en todas las VMs:**
   ```bash
   # Revisar VMs con VBDs sin VDI válido
   for vm in $(xe vm-list is-control-domain=false --minimal | tr ',' ' '); do
       xe vbd-list vm-uuid=$vm params=uuid,vdi-uuid,empty
   done
   ```

3. **Establecer backups regulares de YaeServer** (si no existen)

### Medio Plazo (1-3 meses)
1. **Política de reinicios preventivos:**
   - Reiniciar hypervisor cada 30-60 días durante ventanas de mantenimiento
   - Evita acumulación de problemas de memoria/recursos

2. **Actualización de XCP-ng:**
   - Versión actual: 8.3.0 (Sep 2024)
   - Verificar si existen parches/actualizaciones relacionados con blktap

3. **Implementar alta disponibilidad:**
   - Considerar pool de XCP-ng con múltiples hosts
   - Live migration para mantenimientos sin downtime

### Largo Plazo
1. **Migración a almacenamiento compartido:**
   - Considera NFS/iSCSI SR para facilitar migración entre hosts
   - Actualmente todo en Local storage (LVM)

2. **Implementar monitoreo proactivo:**
   - Nagios/Zabbix para métricas de XCP-ng
   - Alertas tempranas de problemas de storage/tapdisk

---

## LECCIONES APRENDIDAS

1. **Detección temprana:** No hubo alertas previas al fallo de tapdisk
2. **Acceso a consola:** La falta de iLO/IPMI dificultó troubleshooting del reinicio
3. **Documentación:** Este incidente resalta la importancia de procedimientos de respaldo
4. **Downtime:** Un problema de 1 VM requirió reinicio que afectó a todas las VMs

---

## ANEXOS

### Anexo A: Comandos de Diagnóstico Utilizados
```bash
# Verificación de VM
xe vm-list name-label=YaeServer params=all
xe vbd-list vm-uuid=<UUID> params=all

# Verificación de Storage
xe sr-list uuid=<SR-UUID> params=all
xe vdi-list uuid=<VDI-UUID> params=all
vhd-util check -n /dev/VG_XenStorage-.../VHD-xxx

# Diagnóstico de tapdisk
tap-ctl list
tap-ctl spawn
tap-ctl create -a vhd:<path>
ps aux | grep tapdisk
lsof | grep VHD-xxx

# Verificación de kernel/sistema
dmesg | grep -i tapdisk
lsmod | grep blktap
ulimit -a
cat /proc/sys/fs/file-nr
```

### Anexo B: UUIDs de Referencia
- **VM YaeServer:** 1fda53a2-a31b-61ca-e419-0866cb44b134
- **VDI Principal:** 46cca168-69a4-4cd5-86fd-2f9ff35f10c3
- **SR Local storage:** 58956d9f-dfae-9ef2-9ef7-87b531468267
- **VBD eliminado (CD-ROM):** ad97ae89-f38d-0953-5381-626eb3fe7bba

---

**Reporte generado:** 22 de Junio de 2026
**Ingeniero responsable:** Análisis remoto vía SSH
**Estado:** RESUELTO ✓
