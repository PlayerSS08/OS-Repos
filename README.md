# OS CA's – Operating Systems (25PCCS201), research topic R19

One folder per CA; **each folder is a self-contained project in its own GitHub repository**, linked here as a git submodule.

| Folder | CA | Status |
|---|---|---|
| [`CA1_Systems_Thinking`](https://github.com/PlayerSS08/CA1_Systems_Thinking) | CA-1 | Systems Thinking Challenge: F1 high page fault rate (report A–K, stdlib code, 9 tests) |
| [`CA2_Performance_Investigation`](https://github.com/PlayerSS08/CA2_Performance_Investigation) | CA-2 | complete: code, results, 23-page report |
| [`CA3_Design_and_Debugging`](https://github.com/PlayerSS08/CA3_Design_and_Debugging) | CA-3 | complete: code, results, report |
| [`CA4_Research_Mini_Project`](https://github.com/PlayerSS08/CA4_Research_Mini_Project) | CA-4 | your research report (PDF) |
| [`CA5_Linux_Systems_Engineering`](https://github.com/PlayerSS08/CA5_Linux_Systems_Engineering) | CA-5 | complete: code, results, report (Linux needed to re-run) |
| [`CA6_Rust_Micro_OS`](https://github.com/PlayerSS08/CA6_Rust_Micro_OS) | CA-6 | report only; Aeon source must be added |
| [`CA7_Viva_Demonstration`](https://github.com/PlayerSS08/CA7_Viva_Demonstration) | CA-7 | waiting for the brief |

CA-2, CA-3 and CA-5 each contain their own copy of the small shared simulator (`src/`), so any one of them can be cloned and run alone.
Python 3.10+ is enough (`pip install -r requirements.txt`); building the .docx/.pdf additionally needs pandoc and LibreOffice.

## Cloning

```bash
git clone --recurse-submodules https://github.com/PlayerSS08/OS-Repos.git
```

Already cloned without submodules? Run `git submodule update --init`.

## Updating a CA

Commit and push inside the CA folder, then record the new version here:

```bash
git add <CA folder> && git commit -m "Update <CA>" && git push
```
