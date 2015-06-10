# Introduction

`if (!x) ...` => `x || ...`

`if (x) ...` => `x && ...`

# Performance Impact

Negligible on most browsers, though it is somewhat slower on the Android embedded browser.