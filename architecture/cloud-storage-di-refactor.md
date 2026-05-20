# Cloud Storage DI Refactoring (Follow-up)

Deferred from the `nstandif/feat/bun-s3-driver` PR to reduce review burden and merge conflict risk.


## Decision Gate

When S3-compat migration is validated in production:

- **If GCS backend is removed**: skip this refactor entirely — inline `BunS3StorageBackend` into the service or keep it simple.
- **If both backends stay**: proceed with all 4 changes below.

## Why Deferred

- The migration PR is already large — DI restructuring increases review burden and merge conflict risk
- GCS backend is likely temporary (will be removed once S3-compat migration is validated)
- Current if/else backend selection (5 lines) is readable and self-contained
- Test workaround `(service as any).backend` is 1 line, pragmatic, common in NestJS codebases
- Factory provider relocates the conditional logic, doesn't eliminate it

## Changes

### 1. Extract backends to `cloud-storage.backend.ts`

Create `src/cloud-storage/cloud-storage.backend.ts` containing:

- `StorageBackend` abstract class
- `GcsStorageBackend`
- `BunS3StorageBackend`

Remove these from `cloud-storage.service.ts`.

### 2. Factory provider for `StorageBackend`

In `cloud-storage.module.ts`, replace the `S3_CLIENT` factory with:

```typescript
{
  provide: StorageBackend,  // abstract class as DI token (no Symbol needed)
  inject: [ConfigService],
  useFactory(conf: ConfigService<AppConfig, true>): StorageBackend {
    const { bucket, backend, s3 } = conf.get<CloudStorageConfig>("cloudStorage");
    if (backend === "s3") {
      return new BunS3StorageBackend(new S3Client({ ...s3, bucket }));
    }
    return new GcsStorageBackend(new Storage().bucket(bucket));
  },
}
```

In `CloudStorageService`, replace the constructor:

```typescript
constructor(
  conf: ConfigService<AppConfig, true>,
  @Inject(StorageBackend) private readonly backend: StorageBackend,
  private readonly logger: WinstonLoggerService,
) {
  this.bucketName = conf.get<CloudStorageConfig>("cloudStorage").bucket;
}
```

### 3. Remove `S3_CLIENT` token

- Delete `src/cloud-storage/cloud-storage.constants.ts`
- Remove `S3_CLIENT` from module exports

### 4. Update tests

Replace `(service as any).backend = backendMock` with proper DI:

```typescript
providers: [
  CloudStorageService,
  { provide: StorageBackend, useValue: backendMock },
  { provide: ConfigService, useValue: mockConfig },
  { provide: WinstonLoggerService, useValue: mockLogger },
]
```

## References

- [NestJS Custom Providers — abstract class as DI token](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Factory Providers](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)
- Sync factory is correct here — both `S3Client` and `Storage().bucket()` constructors are synchronous
