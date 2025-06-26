+++
title = "Case Sensitivity in GitHub Pages"
date = 2025-06-25T19:37:00-07:00
tags = ["github", "web"]
+++

## TIL: GitHub Pages is Case-Sensitive

Today I learned that GitHub Pages runs on a case-sensitive filesystem, unlike macOS which is case-insensitive by default.

This means that if you have an image file named `Image.jpg` and reference it as `image.jpg` in your markdown:
- It will work locally on macOS
- It will fail with a 404 error on GitHub Pages

### Solution

Always ensure that the case in your file references matches exactly with the actual filenames. This applies to:
- Image paths
- Internal links
- Include files
- Any other file references

If migrating from a case-insensitive system (like Windows or macOS) to GitHub Pages, double-check all your file references.
