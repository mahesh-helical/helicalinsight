# Recycle bin sidecars (import / export)

Soft-deleted files, folders, connections, users, and organizations can travel inside an export package and stay soft-deleted on the destination, including who deleted them and when.

The writer is `com.helicalinsight.export.service.RecycleBinHandler`. In-app restore and purge use a different type: `com.helicalinsight.adhoc.recycle.handler.RecycleBinHandler` (service type `recycleBin`). This page is only the package sidecar.

## Turn the option on

Export and import both send `options.recycleBin`.

| Surface | Control |
|---|---|
| File browser export | Checkbox **Recycle Bin** → `options.recycleBin` (`ExportModal.jsx`) |
| Admin import | Checkbox **Recycle Bin** → `options.recycleBin` (`ImportFiles.jsx`) |
| Server default | `hi-repository/System/Admin/setting.xml` → `importExportOptions/enforceRecycleBin` |

`enforceRecycleBin` is `true` in the shipped `setting.xml`. While it is true the checkbox is forced on. The import tooltip states the other behavior: if Recycle Bin is unselected, imported recycle-bin contents are restored.

Sidecars are written only when the **manifest** option is true (`RecycleBinHandler.isEnabled`). They are applied on import only when `ManifestUtils.compareOptions(..., "recyclebin")` is true: the request option is true, the package option is true, and the two booleans are equal. Either side false or null skips every sidecar.

## What gets a sidecar

Files land under the package `resources/` directory. `Manifest.recycleBins` maps a key to `{key}_recyclebin` (`ResourceSuffix.RECYCLE_BIN`). With manifest version greater than 0, `ResourceDataWriter` keeps the key’s directory path. A key already present in the map is not written again.

| Entity | Call | Manifest key | Written when |
|---|---|---|---|
| File or folder | `writeResource` from the resource writer handlers | Resource path. Folders are also looked up as `{path}.efwfolder` on import. | `HIResource.isDeleted()` is true **and** a bin row exists for that resource id. The resource creator is also offered to `writeUser`. |
| Global connection | `GlobalDSHandler` → `writeGlobalConnection` | `ds_global_{exportGlobalId}` | A bin row exists for that global id |
| EFWD connection | `AdvancedDSHandler` → `writeEfwdConnection` | `ds_efwd_{efwdId}` | A bin row exists for that EFWD id |
| User | `writeUser` from the resource creator and from `ShareHandler` | `user_{userId}` | A bin row exists for that user id |
| Organization | `writeOrganization` from `ShareHandler` | `org_{orgId}` | A bin row exists for that org id |

The JSON is a `RecycleBinDTO`. The writer sets `type`, `recycleBinId`, `deletedOn`, `deletedBy`, and `createdBy`. Null fields are omitted. `deletedBy` and `createdBy` are user objects, not username strings. `type` is one of `HI_RESOURCE_DB`, `DS_GLOBAL_CONNECTIONS`, `H_USERS`, `ORGANIZATION`, `HI_EFWD_CONNECTION`.

Example manifest entry and sidecar:

```json
"recycleBins": {
  "ds_global_12": "ds_global_12_recyclebin"
}
```

```json
{
  "type": "DS_GLOBAL_CONNECTIONS",
  "recycleBinId": 44,
  "deletedOn": "2026-10-22T10:15:00.000+00:00",
  "deletedBy": { "id": 1, "username": "hiadmin" },
  "createdBy": { "id": 1, "username": "hiadmin" }
}
```

## Import sequence

1. The resource or connection importer runs first.
2. `ImportManagerContext.recover` undeletes a destination row that is already in the bin, **unless** both recycle-bin options are on and the manifest contains a sidecar key for that row. Resource keys are the path and `{path}.efwfolder`. Global and EFWD preserve-checks use the **destination** id (`ds_global_{destinationGlobalId}`, `ds_efwd_{destinationEfwdId}`).
3. The handler then imports the sidecar.
   - Resources: `ImportResourceManager` calls `importResource` after `importResource(path)`. If that URL is already in the response skip list, the method returns and does not soft-delete.
   - Global connections: `getOrInsertGlobalConnection` calls `importGlobalConnection(exportGlobalId, destinationConnection, context)`.
   - EFWD: `importEfwdConnection(exportEfwdId, ...)`.
   - Users and organizations: `ResourceShareUtils` calls `importUser` / `importOrganization` with the id from the package. Ids `<= 0` are ignored.
4. `importSidecar` reads `resources/{file from the manifest}`.
   - No manifest entry: return, and do not soft-delete.
   - File unreadable: log `[recycleBin] Sidecar unreadable; skipped.`
   - Otherwise soft-delete the destination when `isDeleted` is not already true, then load the bin row that delete created.
   - No bin row after that: log `[recycleBin] No recycle-bin record after soft-delete; skipped.` The entity may already be marked deleted.
   - On success, copy `deletedBy`, `createdBy`, the created-by user’s organization, and `deletedOn` onto that bin row. Log `[recycleBin] Recycle-bin entry imported.`

Global-connection sidecar lookup uses the **export** id (`ds_global_{exportGlobalId}`), then soft-deletes the destination connection that was matched or inserted. Those ids are often different. Do not look up the sidecar with the destination id.

`deletedBy` / `createdBy` are resolved with `ResourceShareUtils.getOrInsertUser`. A user object that cannot be resolved leaves that field unchanged.

## Pitfalls

| What you see | Why |
|---|---|
| Deleted items come back active | Recycle Bin was off on the request or absent on the package. `recover` clears `isDeleted` and removes the destination bin row. |
| Sidecar file is in the zip but nothing is soft-deleted | Manifest `recycleBins` has no key, or `compareOptions` failed because one side has `recycleBin` null/false. |
| Log: sidecar unreadable | JSON under `resources/` does not map to an object node. Import continues without metadata. |
| Log: no recycle-bin record after soft-delete | The delete call did not insert a `HIRecycleBin` row, so `deletedOn` / `deletedBy` were not copied. |
| Global connection imports active, or metadata looks missing | Sidecar key must stay `ds_global_{id from the package}`. `recover`’s preserve check uses the destination id, so a deleted destination whose id differs from the export id is restored first and then soft-deleted again when the export-id sidecar is found. |
| Conflict action SKIP | `importResource` returns before the sidecar when the resource URL is already in `response.skips`. |
| Same key written twice | `writeBin` returns when the key is already in `recycleBins`. |

`markDeleted` on `AbstractResourceWriterHandler` also sets `isDeleted` on the exported resource payload when the export option is on and the source resource is deleted. The sidecar is still what carries bin metadata.
