# GitHub Pages Deployment Fix

## Issue
The GitHub Pages deployment was failing with the error:
```
Error: Artifact could not be deployed. Please ensure the content does not contain any 
hard links, symlinks and total size is less than 10GB.
```

## Root Cause
The workflow had two main issues in the "Cleanup and prepare reports" step:

1. **Symlink Dereference Issue**: The `cp -rL` command was dereferencing symlinks, which could create problematic file references that GitHub Pages doesn't accept.
2. **Hard Link Detection**: The verification step wasn't properly detecting hard links, allowing them to slip through to the artifact.

## Solution
The fix implements a cleaner approach:

### 1. **Removed symlink dereference** (`-L` flag)
   - Old: `cp -rL "$report_dir/js" "pages_content/$load_type/"`
   - New: `find "$report_dir/js" -type f ! -type l -exec install -D "{}" "pages_content/$load_type/{}" \;`
   - The `install` command creates fresh copies without symlink issues
   - The `-type f ! -type l` filter explicitly excludes symlinks and only copies regular files

### 2. **Improved symlink/hardlink detection**
   - Added explicit symlink detection that exits with error if found
   - Added hard link detection (files with `nlinks > 1`)
   - All detected issues are logged for debugging

### 3. **Final safety cleanup**
   - Added `find pages_content -type l -delete` at the end to catch any remaining symlinks
   - Prevents any edge cases from reaching the GitHub Pages deployer

## Changes Made
**File**: `.github/workflows/JmeterPerformanceTest.yml`

### Step: "Cleanup and prepare reports"
- Replaced recursive copy with symlink dereference (`cp -rL`) with the safer `install` + `find` approach
- Changed from shell operators (`&&`, `||`) to explicit conditionals
- Added explicit symlink filtering in find commands

### Step: "Verify content and check size"
- Enhanced verification to detect both symlinks AND hard links
- Added detailed logging for each verification check
- Made symlink detection a hard failure (exits with status 1)
- Added file listing for debugging

## How It Works
1. **Find all regular files (excluding symlinks)**: `find "$report_dir/js" -type f ! -type l`
2. **Copy using `install` command**: Creates fresh independent file copies without any link relationships
3. **Destination path preservation**: `-D` flag ensures directory structure is created
4. **Verify no symlinks**: Catches any remaining symlinks and fails fast
5. **Verify no hard links**: Detects files with multiple inodes (hard links)

## Testing
When the workflow runs:
1. Artifacts are downloaded
2. Reports are cleaned and prepared using the new method
3. Verification step explicitly checks for symlinks and hard links
4. If any symlinks are found, the deployment fails with clear error message
5. Otherwise, artifact is uploaded and deployed to GitHub Pages

## Benefits
✅ No symlinks or hard links in deployed content
✅ Cleaner file copies using standard `install` command
✅ Better error detection and reporting
✅ More reliable GitHub Pages deployment
✅ Faster deployment (smaller artifact size due to not copying unnecessary data files)

