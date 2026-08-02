# 08 · File Uploads & Validation

Letting visitors upload files — profile pictures, documents, CSVs — is one
of the riskiest things a web app does: the file's name, declared type, and
even its extension are all things an attacker fully controls. This module
covers the `$_FILES` superglobal and, more importantly, how to validate an
upload so a malicious file can't be smuggled onto your server as something
it isn't.

## The upload form

```html
<!-- The form MUST specify enctype="multipart/form-data" -- without it,
     the file's binary content is never actually sent to the server. -->
<form method="post" action="upload.php" enctype="multipart/form-data">
    <input type="file" name="avatar">
    <button type="submit">Upload</button>
</form>
```

## The `$_FILES` superglobal

```php
<?php
// After the form above is submitted, $_FILES looks like this:
print_r($_FILES);
// Array
// (
//     [avatar] => Array
//         (
//             [name] => photo.jpg          // original filename -- NEVER trust this
//             [type] => image/jpeg         // client-reported MIME type -- NEVER trust this either
//             [tmp_name] => /tmp/phpXXXXXX // where PHP actually stored the upload
//             [error] => 0                 // 0 means UPLOAD_ERR_OK
//             [size] => 204800             // bytes
//         )
// )
```

Both `name` and `type` come straight from the browser's request — a
malicious client can set `type` to `image/jpeg` while uploading a PHP
script, or name a file `../../etc/passwd`. Neither field is safe to trust
without independent verification, which the rest of this module covers.

## Step 1: check the upload error code

```php
<?php

function uploadErrorMessage(int $code): ?string
{
    return match ($code) {
        UPLOAD_ERR_OK => null,                                   // no error
        UPLOAD_ERR_INI_SIZE, UPLOAD_ERR_FORM_SIZE => "File is too large",
        UPLOAD_ERR_PARTIAL => "File was only partially uploaded",
        UPLOAD_ERR_NO_FILE => "No file was uploaded",
        UPLOAD_ERR_NO_TMP_DIR, UPLOAD_ERR_CANT_WRITE => "Server storage error",
        default => "Unknown upload error",
    };
}

$error = $_FILES["avatar"]["error"] ?? UPLOAD_ERR_NO_FILE;
if (($message = uploadErrorMessage($error)) !== null) {
    exit("Upload failed: $message");
}
```

Checking `error` first matters because on any failure, `tmp_name` and `size`
may be missing, zero, or meaningless — validating file content before
checking `error` risks working with garbage data from a failed upload.

## Step 2: validate the actual size

```php
<?php

const MAX_UPLOAD_BYTES = 2 * 1024 * 1024;   // 2 MB

$size = $_FILES["avatar"]["size"];
if ($size > MAX_UPLOAD_BYTES) {
    exit("File exceeds the 2 MB limit.");
}
```

This is a second check even though `php.ini`'s `upload_max_filesize` also
caps upload size — the `.ini` limit is a blunt, server-wide default, while
your own check enforces the limit that's actually appropriate for *this*
form field (an avatar and a document import shouldn't share one ceiling).

## Step 3: validate the real file content, not the client's claims

Never trust `$_FILES[...]["type"]` or the file's extension — both are just
text the client sent. Instead, inspect the file's actual bytes with the
`fileinfo` extension, which recognizes real file signatures ("magic bytes")
regardless of what the upload claims to be.

```php
<?php

function detectRealMimeType(string $tmpPath): string
{
    // The object-oriented finfo API needs no explicit close -- the
    // resource is released automatically when $finfo goes out of scope.
    $finfo = new finfo(FILEINFO_MIME_TYPE);
    return $finfo->file($tmpPath);
}

$allowedTypes = ["image/jpeg", "image/png", "image/webp"];

$realType = detectRealMimeType($_FILES["avatar"]["tmp_name"]);
if (!in_array($realType, $allowedTypes, true)) {
    exit("Unsupported file type: $realType");
}
```

A file renamed from `payload.php` to `photo.jpg` still contains PHP source
code as its actual bytes — `finfo::file()` reports its true content type
(e.g. `text/x-php` or `text/plain`), catching the disguise that checking the
extension alone would miss entirely.

## Step 4: generate a safe filename yourself

Never reuse the client-supplied filename for the stored file — it can
contain path traversal sequences (`../../`), null bytes, or characters your
filesystem handles unexpectedly. Generate your own name and keep only a
known-safe extension.

```php
<?php

function safeExtensionFor(string $mimeType): string
{
    return match ($mimeType) {
        "image/jpeg" => "jpg",
        "image/png" => "png",
        "image/webp" => "webp",
        default => throw new InvalidArgumentException("Unsupported type"),
    };
}

$extension = safeExtensionFor($realType);
$safeName = bin2hex(random_bytes(16)) . "." . $extension;   // e.g. 5f3a...c9.jpg
$destination = __DIR__ . "/uploads/" . $safeName;

// move_uploaded_file() (not rename() or copy()) verifies the source path
// really was created by PHP's own upload mechanism -- rejecting attempts
// to trick your script into "uploading" an arbitrary server-side file.
if (!move_uploaded_file($_FILES["avatar"]["tmp_name"], $destination)) {
    exit("Failed to save uploaded file.");
}

echo "Saved as: $safeName\n";
```

The random filename also sidesteps a subtler bug: two visitors uploading
files both named `photo.jpg` at the same moment would otherwise silently
overwrite each other.

## Where NOT to store uploads

Store uploaded files **outside** the web root, or in a directory the web
server is configured not to execute scripts from. If `uploads/` is
publicly reachable at `https://example.com/uploads/` and your validation
has any gap, an attacker who slips a `.php` file past it can have the
server execute it directly just by requesting its URL — turning a file
upload bug into full remote code execution. Serving uploaded files back
through a PHP script that streams the file's bytes (checking permissions on
the way) avoids this even if a bad file ends up stored.

## File upload validation cheat sheet

| Check | Why |
|-------|-----|
| `$_FILES[...]["error"] === UPLOAD_ERR_OK` | Confirms the upload actually completed before touching other fields |
| Your own size check | Enforces a per-form limit, independent of `php.ini` defaults |
| `finfo::file()` on the real bytes | The client's declared `type`/extension can be anything |
| Generate your own filename | Avoids path traversal, null bytes, and overwrite collisions |
| `move_uploaded_file()`, never `rename()` | Verifies the source was a genuine PHP upload |
| Store outside the web root | A validation gap can't become remote code execution |

## Exercise

Write a function `handleAvatarUpload(array $file): string` (accepting one
`$_FILES["avatar"]`-shaped array) that runs all four validation steps above
in order — error code, size limit, real MIME type via `finfo`, then a
generated safe filename — returning the new filename on success or throwing
a descriptive `InvalidArgumentException` at the first failed check. Test it
by hand-building a few different `$file` arrays: one with `UPLOAD_ERR_OK`
pointing at a real small image, one exceeding your size limit, and one
pointing at a plain `.txt` file renamed with a `.jpg` extension.
