## Coverage check

- **P2-T1 — Pass**  
  Defuddle was added and wired into `extractWebArticleFromCurrentPage()` in the expected order (`site spec → discourse OP → defuddle → readability → fallback`).  
  Evidence: `webclipper/src/collectors/web/article-extract/defuddle.ts`, `.../engine.ts`, commit `070f6e19`.

- **P2-T2 — Pass**  
  HTML cleaning and URL absolutization are implemented as fragment-level preprocessing (`cleanHtmlFragment`) and used before Turndown conversion.  
  Evidence: `webclipper/src/collectors/web/article-extract/html-clean.ts`, `.../markdown-turndown.ts`, commit `13ce626b`.

- **P2-T3 — Pass**  
  DOM stabilization wait + site hydration wait are present, and extraction order matches plan intent. Complex markdown regression test added and passing.  
  Evidence: `webclipper/src/collectors/web/article-extract/engine.ts`, `webclipper/tests/smoke/article-extract-markdown-complex.test.ts`, commit `9b667079`.

- **P2-T4 — Pass**  
  Site-spec output now provides HTML/text, with markdown generated centrally in engine via Turndown path (no local hand-built markdown path retained).  
  Evidence: `.../site-spec-extractor.ts`, `.../engine.ts`, updated bilibili/xiaohongshu tests, commit `9dc0ced8`.

- **P2-T5 — Pass**  
  WeChat gallery markdown bypass API removed; gallery remains HTML-appended and goes through unified Turndown conversion path.  
  Evidence: `.../sites/wechat.ts`, `.../sites/index.ts`, `.../engine.ts`, `article-extract-wechat-gallery.test.ts`, commit `2058d7f2`.

## Findings

No blocking findings.

## Suggested fixes

- No fixes required based on current audit scope and evidence.

## Verification notes

Commands run (all succeeded with exit code 0):

1. `git --no-pager status` (working tree clean)
2. `git --no-pager show` on commits `070f6e19`, `13ce626b`, `9b667079`, `9dc0ced8`, `2058d7f2` (task-to-implementation trace)
3. `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
4. Targeted regressions:
   - `npm --prefix webclipper run test -- article-extract-discourse-op-onebox-regression`
   - `npm --prefix webclipper run test -- article-extract-bilibili-opus`
   - `npm --prefix webclipper run test -- article-extract-xiaohongshu`
   - `npm --prefix webclipper run test -- article-extract-wechat-gallery`
   - `npm --prefix webclipper run test -- article-extract-markdown-complex`

Observed summaries in targeted run logs show all selected test files passed.
