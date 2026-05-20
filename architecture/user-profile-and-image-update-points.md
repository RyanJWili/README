# User Profile & Image Update Points

Every location in the codebase where a user's profile data or images are mutated.

---

## Image Updates

### Mobile Upload Service (`src/user/mobile-upload.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `addImagesWithUser` | 315 | Uploads profile or dating-profile images to cloud storage, replaces images via `profileClient.replaceImages`, triggers image classification and profile analysis |
| `deleteImagesWithUser` | 379 | Deletes specified profile images from cloud storage and updates profile via `profileClient.replaceImages` |
| `addIdealDateImagesWithUser` | 415 | Uploads ideal-date images to cloud storage and replaces via `profileClient.replaceIdealDateImages` |
| `deleteIdealImagesWithUser` | 458 | Deletes specified ideal-date images from cloud storage and updates via `profileClient.replaceIdealDateImages` |
| `addImagesWithTempToken` | 496 | Wraps `addImagesWithUser` for temporary-token-based uploads from mobile web |
| `completePhotoSliceUpload` | 207 | Reassembles chunked upload, delegates to `addImagesWithUser` or `addIdealDateImagesWithUser` |
| `finishUpload` | 520 | (deprecated) Calls `profileClient.addImages` |
| `autofillProfile` | 541 | Calls `profileClient.autofillProfile` (may set images) |

### User Controller — Image Endpoints (`src/user/user.controller.ts`)

| Endpoint | Line | Delegates to |
|---|---|---|
| `POST /profile/photo` | 208 | `mobileUploadSvc.addImagesWithUser` |
| `DELETE /profile/photo` | 268 | `mobileUploadSvc.deleteImagesWithUser` |
| `POST /profile/photo/ideal-date` | 273 | `mobileUploadSvc.addIdealDateImagesWithUser` |
| `DELETE /profile/photo/ideal-date` | 291 | `mobileUploadSvc.deleteIdealImagesWithUser` |
| `POST /profile/photo/slice/complete` | 321 | `mobileUploadSvc.completePhotoSliceUpload` |

### Chatbot Tools (`src/chatbot/chatbot-tools.ts`)

| Tool | Line | What it does |
|---|---|---|
| `updateUserProfileImage` | 591 | Downloads image URLs, uploads to cloud storage, calls `llmToolsService.updateUserProfileImages` |
| `deleteProfileImages` | 687 | Deletes profile photos by index via `llmToolsService.deleteProfileImages` |

### LLM Tools Service (`src/llm-tools/llm-tools.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `updateUserProfileImages` | 859 | Uploads images to cloud storage, calls `profileClient.addImages` + triggers image classification |
| `updateUserIdealDateImages` | 876 | Uploads ideal-date images, calls `profileClient.addIdealDateImages` + triggers image classification |
| `deleteProfileImages` | 988 | Deletes images from cloud storage, calls `profileClient.replaceImages` with remaining images |

### Internal User Service (`src/internal/user/internal-user.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `uploadUserPhoto` | 1326 | Uploads images to cloud storage, calls `profileClient.addImages` + triggers image classification |
| `removeProfileImages` | 862 | Deletes all user images from cloud storage (called by `resetUserProfile`) |
| `resetUserProfile` | 903 | Calls `removeProfileImages` + `profileClient.resetProfile` |

### User Service — Account Lifecycle (`src/user/user.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `deleteUser` | 1264 | Deletes all images from cloud storage via `cloudStorageService.deleteWithPrefix`, deletes profile via `profileClient.deleteProfile`, deletes `userImage` docs |
| `mergeAccount` | 1120–1150 | Copies images, dating images, and ideal-date images from source user to target user via `profileClient.replaceImages` / `profileClient.replaceIdealDateImages` |

### Image Classification (`src/image-classification/image-classification.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `generateImageDescriptions` | 175 | Generates AI descriptions for a user's images (writes to profile). Triggered by image uploads in `MobileUploadService`, `LlmToolsService`, `InternalUserService` |

---

## Profile Data Updates

### User Service (`src/user/user.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `signup` | 619 | Sets initial name via `profileClient.updateSelfProfile` on new user creation |
| `applyYikYakAnswersToProfileInfo` | 470 | Maps Yik Yak answers to basicInfo/deepInfo via `profileClient.updateBasicInfo` and `profileClient.updateDeepInfo` |
| `updateSelfProfile` | 1330 | (deprecated) Updates profile via `profileClient.updateSelfProfile` |
| `updateExpectedProfile` | 1341 | (deprecated) Updates expected profile via `profileClient.updateExpectedProfile` |
| `updateBasicInfo` | 1357 | Updates basic profile fields via `profileClient.updateBasicInfo`, triggers profile analysis and hobby matching |
| `updateDeepInfo` | 1422 | Updates deep profile fields via `profileClient.updateDeepInfo` |
| `updatePoolInfo` | 1435 | Updates pool-specific profile data via `profileClient.updatePoolInfo` |
| `updateExternalMetadata` | 1451 | Updates external metadata via `profileClient.updateExternalMetadata` |
| `calculateOnboardStep` | 672 | Updates `onboardStep2` field on user document |
| `updateOnboardStep` | 1523 | Updates `onboardStep` field on user document |
| `updateOnboardStep2` | 1536 | Updates `onboardStep2` field on user document |
| `mergeAccount` | 1107–1158 | Copies basicInfo, deepInfo, images, tags from source to target user |
| `deleteUser` | 1273 | Deletes profile via `profileClient.deleteProfile` |
| `_recordLoginIp` | 458 | Updates `lastLoginAt` and `lastLoginIp` on user document |
| `updatePhoneNumber` | 1835 | Updates `phone` on user document |

### User Controller — Profile Endpoints (`src/user/user.controller.ts`)

| Endpoint | Line | Delegates to |
|---|---|---|
| `PUT /profile` | 148 | `userService.updateSelfProfile` (deprecated) |
| `PUT /profile/basic-info` | 166 | `userService.updateBasicInfo` |
| `PUT /profile/deep-info` | 176 | `userService.updateDeepInfo` |
| `PUT /profile/pool-info` | 185 | `userService.updatePoolInfo` |

### ProfileClient (`src/profile-client/profile-client.service.ts`)

All profile mutations flow through `ProfileClient`, which delegates to `ProfileService`:

| Method | Line | What it does |
|---|---|---|
| `updateSelfProfile` | 43 | (deprecated) Updates full profile |
| `updateExpectedProfile` | 52 | (deprecated) Updates expected profile |
| `updateBasicInfo` | 61 | Updates basic info fields + checks underage status + triggers profile embedding task |
| `updateDeepInfo` | 112 | Updates deep info fields |
| `updatePoolInfo` | 118 | Updates pool-specific data |
| `updateExternalMetadata` | 124 | Updates third-party metadata |
| `updateProfileNote` | 133 | Updates internal profile note |
| `updateTags` | 227 | Updates profile tags |
| `updateIdentityScale` | 233 | Updates identity scale values |
| `resetProfile` | 189 | Resets profile to empty state |
| `autofillProfile` | 195 | Auto-fills profile data |
| `deleteProfile` | 184 | Deletes entire profile |

### Chatbot Tools (`src/chatbot/chatbot-tools.ts`)

| Tool | Line | Delegates to |
|---|---|---|
| `updateUserProfile` | 348 | `llmToolsService.updateUserProfile` → updates basicInfo and deepInfo |

### LLM Tools Service (`src/llm-tools/llm-tools.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `updateUserProfile` | 462 | Updates basicInfo and deepInfo via `userService.updateBasicInfo` / `userService.updateDeepInfo` |
| `unsetDeepInfoFields` | 588 | Clears deep info fields via `profileClient.updateDeepInfo` |

### Internal User Service (`src/internal/user/internal-user.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `updateNote` | 786 | Updates profile note via `profileClient.updateProfileNote` |
| `resetUserProfile` | 903 | Resets onboardStep + profile via `profileClient.resetProfile` |
| `updateIdentityScale` | 1046 | Updates identity scale via `profileClient.updateIdentityScale` |
| `updateTags` | 1050 | Updates tags via `profileClient.updateTags` |
| `updateProfileField` | 1054 | Updates specific basic info fields (birthday, gender, ethnicity, height) via `profileClient.updateBasicInfo` |
| `verifyEmail` | 1683 | Updates `email`, `emailVerified`, `school`, `inWaitlist` on user document |
| `updateStatus` (batch) | 768–773 | Updates pool arrays on user documents (adds/removes pool codes) |
| `setAccountStatus` | account.service.ts:15 | Sets `accountStatus` on user document |
| `removeAccountStatus` | account.service.ts:25 | Unsets `accountStatus` on user document |
| `updateExternalMetadata` | controller:375 | Updates external metadata via `userService.updateExternalMetadata` |

### Internal Pool Service (`src/internal/pool/internal-pool.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `moveUserPool` | 207 | Updates `pool` array on user document (swap from one pool to another) |

### Internal School Service (`src/internal/school/internal-school.service.ts`)

| Method | Line | What it does |
|---|---|---|
| `mergeSchools` | 254 | Updates `school` field on all users from source school to target school |
