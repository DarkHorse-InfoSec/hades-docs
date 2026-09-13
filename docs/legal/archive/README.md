# EULA archive

Superseded EULA documents, kept unmodified. Do not edit anything in this folder.

The current EULA is one directory up: `docs/legal/HADES_EULA_v1.2.0.docx`.

| File | MD5 of the original bytes | What it was |
|---|---|---|
| `HADES_EULA_v1.1.0.public.fa8c082436681bded18e61ff4f2cd14c.docx` | `fa8c082436681bded18e61ff4f2cd14c` | The copy that lived in this repository. It said the undetected remainder was 1.3% and that License Keys are signed with HMAC-SHA256. Both were wrong. |

A second file also named `HADES_EULA_v1.1.0.docx`, with the identical cover line
"Version 1.1.0  |  Effective Date: April 2, 2026" but three substantive paragraphs
different, lived in the private `darkhorse-hades` repository and shipped in every build.
That divergence is the defect version 1.2.0 was created to end. Both old files are
archived side by side in the private repository at `docs/legal/archive/`.

The filename carries the MD5 of the file as it was found on 2026-09-13, so it is
checkable that nothing was altered on the way in.

Why it was superseded, paragraph by paragraph: `tasks/eula_v120_changes_20260913.md`
in the private `darkhorse-hades` repository.
