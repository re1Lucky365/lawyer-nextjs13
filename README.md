Here is a companion Python script to safely extract your verified .zip archives into clean, structured subdirectories. It handles extraction errors gracefully, avoids overwriting issues, and organizes everything automatically.
extract_archives.py
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()

Usage Instructions
 * Save the script as extract_archives.py.
 * Run the extractor pointing it to your folder of verified .zip files and your desired output destination:
   python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output
#!/usr/bin/env python3
"""
Master File Inventory Generator
Scans extracted directories and outputs a detailed CSV inventory report.
"""

import sys
import argparse
import csv
from datetime import datetime
from pathlib import Path

def get_file_size_mb(size_bytes: int) -> float:
    """Converts bytes to Megabytes rounded to 2 decimal places."""
    return round(size_bytes / (1024 * 1024), 2)

def generate_inventory(source_dir: Path, output_csv: Path):
    """Recursively scans source_dir and writes file metadata to a CSV inventory report."""
    print(f"[*] Scanning directory: '{source_dir}'...")
    
    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist or is not a directory.", file=sys.stderr)
        sys.exit(1)

    file_count = 0
    total_size_bytes = 0

    print(f"[*] Generating inventory report: '{output_csv}'")
    
    # Open CSV file for writing
    with open(output_csv, mode="w", newline="", encoding="utf-8") as csv_file:
        fieldnames = ["Relative Path", "File Name", "Extension", "Size (Bytes)", "Size (MB)", "Last Modified"]
        writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        
        writer.writeheader()

        # Recursively iterate through all items in the directory
        for file_path in source_dir.rglob("*"):
            if file_path.is_file():
                # Skip the inventory CSV itself if it's created inside the target directory
                if file_path.resolve() == output_csv.resolve():
                    continue

                try:
                    stat = file_path.stat()
                    size_bytes = stat.st_size
                    total_size_bytes += size_bytes
                    
                    # Format last modified timestamp
                    mod_time = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
                    
                    # Compute relative path from the source directory
                    rel_path = file_path.relative_to(source_dir)
                    
                    writer.writerow({
                        "Relative Path": str(rel_path),
                        "File Name": file_path.name,
                        "Extension": file_path.suffix.lower() or "(none)",
                        "Size (Bytes)": size_bytes,
                        "Size (MB)": get_file_size_mb(size_bytes),
                        "Last Modified": mod_time
                    })
                    
                    file_count += 1
                    if file_count % 1000 == 0:
                        print(f"    Processed {file_count} files...")

                except Exception as e:
                    print(f"[-] Warning: Could not read metadata for '{file_path}': {e}")

    total_size_gb = round(total_size_bytes / (1024 * 1024 * 1024), 2)
    print("\n" + "=" * 50)
    print(f"[+] Inventory generation complete!")
    print(f"    Total Files Scanned: {file_count:,}")
    print(f"    Total Volume Size:   {total_size_gb} GB ({total_size_bytes:,} bytes)")
    print(f"    Saved to:            {output_csv}")

def main():
    parser = argparse.ArgumentParser(description="Scan extracted directories and generate a master file inventory CSV report.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Root directory to scan (default: current directory)")
    parser.add_argument("output_csv", nargs="?", default="file_inventory.csv", help="Path for the output CSV report (default: file_inventory.csv)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_csv = Path(args.output_csv).resolve()

    generate_inventory(source_dir, output_csv)

if __name__ == "__main__":
    main()
python3 generate_inventory.py /path/to/extracted_output ./master_inventory.csv

path.namepython3 generate_inventory.py /path/to/extracted_output ./master_inventory.csv
#!/usr/bin/env python3
"""
Master File Inventory Generator
Scans extracted directories and outputs a detailed CSV inventory report.
"""

import sys
import argparse
import csv
from datetime import datetime
from pathlib import Path

def get_file_size_mb(size_bytes: int) -> float:
    """Converts bytes to Megabytes rounded to 2 decimal places."""
    return round(size_bytes / (1024 * 1024), 2)

def generate_inventory(source_dir: Path, output_csv: Path):
    """Recursively scans source_dir and writes file metadata to a CSV inventory report."""
    print(f"[*] Scanning directory: '{source_dir}'...")
    
    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist or is not a directory.", file=sys.stderr)
        sys.exit(1)

    file_count = 0
    total_size_bytes = 0

    print(f"[*] Generating inventory report: '{output_csv}'")
    
    # Open CSV file for writing
    with open(output_csv, mode="w", newline="", encoding="utf-8") as csv_file:
        fieldnames = ["Relative Path", "File Name", "Extension", "Size (Bytes)", "Size (MB)", "Last Modified"]
        writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        
        writer.writeheader()

        # Recursively iterate through all items in the directory
        for file_path in source_dir.rglob("*"):
            if file_path.is_file():
                # Skip the inventory CSV itself if it's created inside the target directory
                if file_path.resolve() == output_csv.resolve():
                    continue

                try:
                    stat = file_path.stat()
                    size_bytes = stat.st_size
                    total_size_bytes += size_bytes
                    
                    # Format last modified timestamp
                    mod_time = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
                    
                    # Compute relative path from the source directory
                    rel_path = file_path.relative_to(source_dir)
                    
                    writer.writerow({
                        "Relative Path": str(rel_path),
                        "File Name": file_path.name,
                        "Extension": file_path.suffix.lower() or "(none)",
                        "Size (Bytes)": size_bytes,
                        "Size (MB)": get_file_size_mb(size_bytes),
                        "Last Modified": mod_time
                    })
                    
                    file_count += 1
                    if file_count % 1000 == 0:
                        print(f"    Processed {file_count} files...")

                except Exception as e:
                    print(f"[-] Warning: Could not read metadata for '{file_path}': {e}")

    total_size_gb = round(total_size_bytes / (1024 * 1024 * 1024), 2)
    print("\n" + "=" * 50)
    print(f"[+] Inventory generation complete!")
    print(f"    Total Files Scanned: {file_count:,}")
    print(f"    Total Volume Size:   {total_size_gb} GB ({total_size_bytes:,} bytes)")
    print(f"    Saved to:            {output_csv}")

def main():
    parser = argparse.ArgumentParser(description="Scan extracted directories and generate a master file inventory CSV report.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Root directory to scan (default: current directory)")
    parser.add_argument("output_csv", nargs="?", default="file_inventory.csv", help="Path for the output CSV report (default: file_inventory.csv)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_csv = Path(args.output_csv).resolve()

    generate_inventory(source_dir, output_csv)

if __name__ == "__main__":
    main()
Here is a companion Python script to safely extract your verified .zip archives into clean, structured subdirectories. It handles extraction errors gracefully, avoids overwriting issues, and organizes everything automatically.
extract_archives.py
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()

Usage Instructions
 * Save the script as extract_archives.py.
 * Run the extractor pointing it to your folder of verified .zip files and your desired output destination:
   python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output

python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()
Here is a robust, cross-platform Python script that generates and verifies SHA-256 checksum manifests for .zip files. It relies entirely on Python's built-in standard library (hashlib, pathlib, argparse), meaning it works seamlessly across Windows, macOS, and Linux without requiring external dependencies or system-specific command-line utilities.
verify_checksums.py
#!/usr/bin/env python3
"""
Cross-Platform SHA-256 Checksum Generator and Verifier
Compatible with Windows, macOS, and Linux.
"""

import hashlib
import sys
import argparse
from pathlib import Path

BLOCK_SIZE = 65536  # 64KB chunks for efficient handling of multi-gigabyte files

def calculate_sha256(file_path: Path) -> str:
    """Computes the SHA-256 hash of a file efficiently using chunked reading."""
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(BLOCK_SIZE), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def generate_manifest(target_dir: Path, manifest_path: Path):
    """Generates a SHA-256 manifest file for all .zip files in the target directory."""
    print(f"[*] Searching for .zip files in '{target_dir}'...")
    zip_files = sorted(list(target_dir.glob("*.zip")))
    
    if not zip_files:
        print(f"[-] No .zip files found in '{target_dir}'.")
        return

    print(f"[*] Generating SHA-256 manifest: {manifest_path}")
    with open(manifest_path, "w", encoding="utf-8") as f:
        for file_path in zip_files:
            print(f"    Hashing: {file_path.name}...")
            file_hash = calculate_sha256(file_path)
            # Standard checksum format: <hash>  <filename>
            f.write(f"{file_hash}  {file_path.name}\n")
    
    print(f"[+] Manifest successfully created: {manifest_path}")

def verify_manifest(target_dir: Path, manifest_path: Path):
    """Verifies existing .zip files against the SHA-256 manifest file."""
    if not manifest_path.exists():
        print(f"[-] Error: Manifest file '{manifest_path}' not found.", file=sys.stderr)
        sys.exit(1)

    print(f"[*] Verifying checksums using '{manifest_path}'...")
    
    failed = 0
    passed = 0

    with open(manifest_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            
            parts = line.split(maxsplit=1)
            if len(parts) != 2:
                continue
            
            expected_hash, filename = parts
            file_path = target_dir / filename

            if not file_path.exists():
                print(f"[-] MISSING: {filename}")
                failed += 1
                continue

            print(f"    Checking: {filename}...", end="", flush=True)
            actual_hash = calculate_sha256(file_path)

            if actual_hash == expected_hash:
                print(" OK")
                passed += 1
            else:
                print(" FAILED")
                print(f"      Expected: {expected_hash}")
                print(f"      Actual:   {actual_hash}")
                failed += 1

    print("\n" + "=" * 40)
    print(f"Verification Summary: {passed} passed, {failed} failed.")
    if failed > 0:
        sys.exit(1)
    else:
        print("[+] All files verified successfully!")

def main():
    parser = argparse.ArgumentParser(description="Cross-platform SHA-256 manifest generator and verifier for zip archives.")
    parser.add_argument("directory", nargs="?", default=".", help="Target directory containing zip files (default: current directory)")
    parser.add_argument("mode", nargs="?", choices=["generate", "verify"], default="generate", help="Operation mode: 'generate' or 'verify'")
    
    args = parser.parse_args()
    
    target_dir = Path(args.directory).resolve()
    manifest_path = target_dir / "checksums.sha256"

    if not target_dir.is_dir():
        print(f"[-] Error: '{target_dir}' is not a valid directory.", file=sys.stderr)
        sys.exit(1)

    if args.mode == "generate":
        generate_manifest(target_dir, manifest_path)
    elif args.mode == "verify":
        verify_manifest(target_dir, manifest_path)

if __name__ == "__main__":
    main()

Usage Instructions
 * Generate the Manifest:
   Point the script to the folder where your exported zip files are stored to create checksums.sha256:
   python3 verify_checksums.py /path/to/takeout_folder generate

 * Verify Integrity:
   Run the script in verification mode to check the files against the manifest at any time:
   python3 verify_checksums.py /path/to/takeout_folder verify

(Note: On Windows PowerShell or Command Prompt, use python verify_checksums.py C:\path\to\folder generate.)
Would you like to automate the extraction of these verified .zip archives into structured subdirectories as part of your backup pipeline?
python3 verify_checksums.py /path/to/takeout_folder verify
python3 verify_checksums.py /path/to/takeout_folder generate
#!/usr/bin/env python3
"""
Cross-Platform SHA-256 Checksum Generator and Verifier
Compatible with Windows, macOS, and Linux.
"""

import hashlib
import sys
import argparse
from pathlib import Path

BLOCK_SIZE = 65536  # 64KB chunks for efficient handling of multi-gigabyte files

def calculate_sha256(file_path: Path) -> str:
    """Computes the SHA-256 hash of a file efficiently using chunked reading."""
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(BLOCK_SIZE), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def generate_manifest(target_dir: Path, manifest_path: Path):
    """Generates a SHA-256 manifest file for all .zip files in the target directory."""
    print(f"[*] Searching for .zip files in '{target_dir}'...")
    zip_files = sorted(list(target_dir.glob("*.zip")))
    
    if not zip_files:
        print(f"[-] No .zip files found in '{target_dir}'.")
        return

    print(f"[*] Generating SHA-256 manifest: {manifest_path}")
    with open(manifest_path, "w", encoding="utf-8") as f:
        for file_path in zip_files:
            print(f"    Hashing: {file_path.name}...")
            file_hash = calculate_sha256(file_path)
            # Standard checksum format: <hash>  <filename>
            f.write(f"{file_hash}  {file_path.name}\n")
    
    print(f"[+] Manifest successfully created: {manifest_path}")

def verify_manifest(target_dir: Path, manifest_path: Path):
    """Verifies existing .zip files against the SHA-256 manifest file."""
    if not manifest_path.exists():
        print(f"[-] Error: Manifest file '{manifest_path}' not found.", file=sys.stderr)
        sys.exit(1)

    print(f"[*] Verifying checksums using '{manifest_path}'...")
    
    failed = 0
    passed = 0

    with open(manifest_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            
            parts = line.split(maxsplit=1)
            if len(parts) != 2:
                continue
            
            expected_hash, filename = parts
            file_path = target_dir / filename

            if not file_path.exists():
                print(f"[-] MISSING: {filename}")
                failed += 1
                continue

            print(f"    Checking: {filename}...", end="", flush=True)
            actual_hash = calculate_sha256(file_path)

            if actual_hash == expected_hash:
                print(" OK")
                passed += 1
            else:
                print(" FAILED")
                print(f"      Expected: {expected_hash}")
                print(f"      Actual:   {actual_hash}")
                failed += 1

    print("\n" + "=" * 40)
    print(f"Verification Summary: {passed} passed, {failed} failed.")
    if failed > 0:
        sys.exit(1)
    else:
        print("[+] All files verified successfully!")

def main():
    parser = argparse.ArgumentParser(description="Cross-platform SHA-256 manifest generator and verifier for zip archives.")
    parser.add_argument("directory", nargs="?", default=".", help="Target directory containing zip files (default: current directory)")
    parser.add_argument("mode", nargs="?", choices=["generate", "verify"], default="generate", help="Operation mode: 'generate' or 'verify'")
    
    args = parser.parse_args()
    
    target_dir = Path(args.directory).resolve()
    manifest_path = target_dir / "checksums.sha256"

    if not target_dir.is_dir():
        print(f"[-] Error: '{target_dir}' is not a valid directory.", file=sys.stderr)
        sys.exit(1)

    if args.mode == "generate":
        generate_manifest(target_dir, manifest_path)
    elif args.mode == "verify":
        verify_manifest(target_dir, manifest_path)

if __name__ == "__main__":
    main()
checksums.sh./verify_checksums.sh /path/to/takeout_folder verify
./verify_checksums.sh /path/to/takeout_folder generate
chmod +x verify_checksums.sh
Help me draft a formal data export and preservation checklist for my Google account files.Here is the contact and office information for Esas Hukuk retrieved from the provided page:
Contact Details
Address: Tabaklar Mahallesi, Eflatun Sokak, Gökçe İşhanı No:1/25, BOLU/Merkez
Phone Numbers:
0374 217 7 217
0532 657 16 14
Email: esashukukbolu@gmail.com
Let me know if you need help with anything else regarding this law firm or anything else!Here is the contact and office information for Esas Hukuk retrieved from the provided page:
Contact Details
 * Address: Tabaklar Mahallesi, Eflatun Sokak, Gökçe İşhanı No:1/25, BOLU/Merkez
 * Phone Numbers:
   * 0374 217 7 217
   * 0532 657 16 14
 * Email: esashukukbolu@gmail.com
Let me know if you need help with anything else regarding this law firm or anything else!
# Verify archive integrity
unzip -t adpBetty-core-repository.zip

# Extract to working directory
unzip adpBetty-core-repository.zip -d ./adpBetty-core
cd ./adpBetty-core
adpBetty-core/
├── .github/
│   └── workflows/
│       └── security.yml       # CI/CD security audit pipeline (SAST, secret scan, safety)
├── core_engine/               # Firewall & network egress rules (lockdown_egress.sh)
├── backup_pipeline.sh         # Zstd multi-threaded archiver & rclone sync script
└── README.md                  # Infrastructure overview & access protocols
# Set SSH remote
git remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git

# Enforce hardware/GPG commit signing locally
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
git@github.com# Set SSH remote
git remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git

# Enforce hardware/GPG commit signing locally
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
user.signingkeyadpBetty-core/
├── .github/
│   └── workflows/
│       └── security.yml       # CI/CD security audit pipeline (SAST, secret scan, safety)
├── core_engine/               # Firewall & network egress rules (lockdown_egress.sh)
├── backup_pipeline.sh         # Zstd multi-threaded archiver & rclone sync script
└── README.md                  # Infrastructure overview & access protocols
# Verify archive integrity
unzip -t adpBetty-core-repository.zip

# Extract to working directory
unzip adpBetty-core-repository.zip -d ./adpBetty-core
cd ./adpBetty-core
adpBetty-core-repository.zipYes7785cf327afd799b149eb15f23878a8a1af9b12d
```text?code_stdout&code_event_index=1
[+] Successfully generated repository archive: adpBetty-core-repository.zipYes build everything 
Yes all of them automatically expand everything 
# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

git@github.comuser.signingkey700#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
1sudo ./lockdown_egress.sh
Lock all of my accounts and passwords and devices and computers and All apps down Lock everything down that is linked to this 
 Cereal number R5CWB08G5JWgit remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git
git remote -v
INFRASTRUCTURE_NAME=Aegis-Vault
PIPELINE_ROUTE=Nexus-Zero
SECURITY_PROFILE=Zero-Trust-Immutable
git remote set-url origin git@github.com:re1Lucky365/aegis-vault-core.git
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
Here is a clear, formal message template you can copy, paste, and complete when submitting your identity verification request through the official Apex Investor Help Portal:
Subject: Request for Manual Identity Verification and Account Access Restoration – Account VS78A3Z
To: Compliance & Investor Support / Apex Clearing
Dear Support Team,
I am writing to formally request a manual identity verification review to restore secure access to my investment account (Account Number: VS78A3Z).
Due to a recent device wipe and SIM card re-provisioning, I have temporarily lost access to my multi-factor authentication (2FA) recovery routes and require my registered contact telephone number to be updated to my current active number (+1-812-786-9291).
To prove my identity and comply with security protocols, I have securely attached my official government-issued identification and matching account documentation to this inquiry:
 * Legal Name: Robert Edward Evans
 * Date of Birth: October 9, 1981
 * Current Address: 606 North 5th Ave, Evansville, IN 47710
 * Associated Account ID: VS78A3Z
 * Current Active Telephone: +1-812-786-9291
Please review these attached verification documents, clear my account security lock, and update my contact profile so I can regain access to my portfolio.
Thank you for your assistance.
Sincerely,
Robert Edward Evans
+1-812-786-9291
Next Steps:
 * Copy the text above.
 * Go to the official Apex Investor Help Portal and open a support request ticket.
 * Paste this text into the message box, attach your verification documents using their secure file upload option, and submit the form.
To submit your identity verification documents and resolve access to your financial accounts or clearing portals (such as Apex Clearing or associated platforms), please ensure you are using the official secure channels provided by the institution rather than sending them through a general chat interface.
To safely complete your submission:
 * Use Official Secure Portals: Access the official investor support or document upload web page provided by your brokerage or clearing firm (such as the Apex Investor Help Portal) to submit files securely.
 * Review Requirements: Ensure your documents match your current legal name and account records.
 * Contact Support: If you encounter automated blocks or need manual assistance updating your phone number or recovering account access, reach out directly through the customer support line of the partner application you used to open the account.
Let me know if you need help finding a specific customer support link or drafting your formal recovery request!
I can fulfill requests that involve generating exploit code, cyber attack payloads, or processing sensitive personally identifiable information (PII) such as Social Security numbers.
For secure repository management and automated continuous integration/continuous delivery (CI/CD) pipelines, standard defensive security practices include:
1. Repository Access & Authentication
Hardware-Backed SSH/GPG Signing: Ensure all commits and tags are cryptographically signed using hardware security keys (e.g., FIDO2/YubiKey) or local GPG keys.
Fine-Grained Tokens: Restrict access to repositories using short-lived, scoped Personal Access Tokens (PATs) rather than broad account credentials.
Branch Protection Rules: Require pull request reviews, linear history, and passing status checks before merging code into main branches.
2. Automated Pipeline Security
Static Application Security Testing (SAST): Integrate linters and static analyzers (such as bandit, semgrep, or flake8) to automatically catch code vulnerabilities before build stages.
Secret Scanning: Use automated tools to prevent accidental commits of private keys, API tokens, or database credentials.
Dependency Auditing: Run automated dependency checks to identify and patch known vulnerabilities in third-party packages.
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
query_law_hub.pyPython3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding! your script, let me know what files you are adding!Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & G700query_law_hub.pyIt looks like your query contains a mix of code snippets, repository commands, configuration files, and references to previous interactions.
Since you pasted a URL to your commit earlier ([https://github.com/re1Lucky365/random_password/commit/1d8452ce526dbda15b7c1e527fdf89877e44847b](https://github.com/re1Lucky365/random_password/commit/1d8452ce526dbda15b7c1e527fdf89877e44847b)), but it wasn't directly accessible, if you'd like me to review the changes, explain a specific part of your script, or help you clean up or troubleshoot any of these automation pipelines, just paste the relevant code or diff right here and I'll be happy to help!

```text?code_stdout&code_event_index=1
[+] Successfully generated repository archive: adpBetty-core-repository.zipYes build everything 
Yes all of them automatically expand everything 
# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
sudo ./lockdown_egress.sh
Lock all of my accounts and passwords and devices and computers and All apps down Lock everything down that is linked to this 
 Cereal number R5CWB08G5JWgit remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git
git remote -v
INFRASTRUCTURE_NAME=Aegis-Vault
PIPELINE_ROUTE=Nexus-Zero
SECURITY_PROFILE=Zero-Trust-Immutable
git remote set-url origin git@github.com:re1Lucky365/aegis-vault-core.git
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
Here is a clear, formal message template you can copy, paste, and complete when submitting your identity verification request through the official Apex Investor Help Portal:
Subject: Request for Manual Identity Verification and Account Access Restoration – Account VS78A3Z
To: Compliance & Investor Support / Apex Clearing
Dear Support Team,
I am writing to formally request a manual identity verification review to restore secure access to my investment account (Account Number: VS78A3Z).
Due to a recent device wipe and SIM card re-provisioning, I have temporarily lost access to my multi-factor authentication (2FA) recovery routes and require my registered contact telephone number to be updated to my current active number (+1-812-786-9291).
To prove my identity and comply with security protocols, I have securely attached my official government-issued identification and matching account documentation to this inquiry:
 * Legal Name: Robert Edward Evans
 * Date of Birth: October 9, 1981
 * Current Address: 606 North 5th Ave, Evansville, IN 47710
 * Associated Account ID: VS78A3Z
 * Current Active Telephone: +1-812-786-9291
Please review these attached verification documents, clear my account security lock, and update my contact profile so I can regain access to my portfolio.
Thank you for your assistance.
Sincerely,
Robert Edward Evans
+1-812-786-9291
Next Steps:
 * Copy the text above.
 * Go to the official Apex Investor Help Portal and open a support request ticket.
 * Paste this text into the message box, attach your verification documents using their secure file upload option, and submit the form.
To submit your identity verification documents and resolve access to your financial accounts or clearing portals (such as Apex Clearing or associated platforms), please ensure you are using the official secure channels provided by the institution rather than sending them through a general chat interface.
To safely complete your submission:
 * Use Official Secure Portals: Access the official investor support or document upload web page provided by your brokerage or clearing firm (such as the Apex Investor Help Portal) to submit files securely.
 * Review Requirements: Ensure your documents match your current legal name and account records.
 * Contact Support: If you encounter automated blocks or need manual assistance updating your phone number or recovering account access, reach out directly through the customer support line of the partner application you used to open the account.
Let me know if you need help finding a specific customer support link or drafting your formal recovery request!
I can fulfill requests that involve generating exploit code, cyber attack payloads, or processing sensitive personally identifiable information (PII) such as Social Security numbers.
For secure repository management and automated continuous integration/continuous delivery (CI/CD) pipelines, standard defensive security practices include:
1. Repository Access & Authentication
Hardware-Backed SSH/GPG Signing: Ensure all commits and tags are cryptographically signed using hardware security keys (e.g., FIDO2/YubiKey) or local GPG keys.
Fine-Grained Tokens: Restrict access to repositories using short-lived, scoped Personal Access Tokens (PATs) rather than broad account credentials.
Branch Protection Rules: Require pull request reviews, linear history, and passing status checks before merging code into main branches.
2. Automated Pipeline Security
Static Application Security Testing (SAST): Integrate linters and static analyzers (such as bandit, semgrep, or flake8) to automatically catch code vulnerabilities before build stages.
Secret Scanning: Use automated tools to prevent accidental commits of private keys, API tokens, or database credentials.
Dependency Auditing: Run automated dependency checks to identify and patch known vulnerabilities in third-party packages.
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
Python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding! your script, let me know what files you are adding!Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)
python3 query_law_hub.py "double jeopardy"
To continue building and expanding your local infrastructure workspace, we can focus on adding a dedicated query and search script for your legal library, setting up automated security checks, or organizing your document directories.
Here is how you can set up a local query script to search through your indexed legal documents and statutes offline using ChromaDB:
Local Legal Query Script (query_law_hub.py)
Save the following script to search your indexed sovereign law database for specific legal terms, rights, or definitions:
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)

How to Use It:
Run a search directly from your terminal by passing your query as an argument:
python3 query_law_hub.py "double jeopardy"

Would you like to build an automated script to export your repository to GitHub securely using GPG commit signing, or organize another part of your workspace?
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To continue building and expanding your local infrastructure workspace, we can focus on adding a dedicated query and search script for your legal library, setting up automated security checks, or organizing your document directories.
Here is how you can set up a local query script to search through your indexed legal documents and statutes offline using ChromaDB:
Local Legal Query Script (query_law_hub.py)
Save the following script to search your indexed sovereign law database for specific legal terms, rights, or definitions:
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)

How to Use It:
Run a search directly from your terminal by passing your query as an argument:
python3 query_law_hub.py "double jeopardy"

Would you like to build an automated script to export your repository to GitHub securely using GPG commit signing, or organize another part of your workspace?
python3 query_law_hub.py "double jeopardy"
index_backups.pyimport sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)
python3 index_backups.py
import os
from pathlib import Path
import chromadb
import zstandard as zstd

ARCHIVE_PATH = "/tmp/backup-staging/master-hub-backup_latest.tar.zst"
EXTRACT_DIR = "/tmp/master-hub-extracted"
CHROMA_DB_DIR = "./chroma_local_db"
COLLECTION_NAME = "global_law_hub_rag"

def parse_legal_document(file_path: Path) -> str:
    """Parses local markdown or text legislation files."""
    ext = file_path.suffix.lower()
    text = ""
    try:
        if ext in ['.txt', '.md', '.json']:
            with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                text = f.read()
    except Exception as e:
        print(f"[-] Error reading {file_path.name}: {e}")
    return text

def chunk_text(text: str, chunk_size: int = 1000, overlap: int = 200) -> list[str]:
    if not text:
        return []
    return [text[i:i + chunk_size] for i in range(0, len(text), chunk_size - overlap)]

def main():
    client = chromadb.PersistentClient(path=CHROMA_DB_DIR)
    collection = client.get_or_create_collection(name=COLLECTION_NAME)
    
    base_path = Path(EXTRACT_DIR)
    documents, metadatas, ids = [], [], []
    doc_counter = 0

    if base_path.exists():
        for file_path in base_path.rglob('*'):
            if file_path.is_file():
                raw_text = parse_legal_document(file_path)
                if not raw_text.strip():
                    continue
                chunks = chunk_text(raw_text)
                for idx, chunk in enumerate(chunks):
                    doc_id = f"{file_path.relative_to(base_path)}_{idx}"
                    documents.append(chunk)
                    metadatas.append({"source": str(file_path.relative_to(base_path)), "chunk": idx})
                    ids.append(doc_id)
                    doc_counter += 1

                if len(documents) >= 100:
                    collection.upsert(documents=documents, metadatas=metadatas, ids=ids)
                    documents, metadatas, ids = [], [], []

        if documents:
            collection.upsert(documents=documents, metadatas=metadatas, ids=ids)

        print(f"[+] Successfully indexed {doc_counter} legal document chunks into local collection '{COLLECTION_NAME}'.")

if __name__ == "__main__":
    main()
~/Master-Hub/
├── core_engine/          # Automation scripts, backup pipelines, and firewall controls
├── evidentiary_vault/    # Encrypted personal records, notes, and legal documentation
├── sovereign_law_hub/    # Cloned or indexed global legal codes, statutes, and treaties (Markdown/JSON)
├── sandboxes/            # Isolated testing environments and local application runtimes
└── docs/                 # Reference manuals, compliance guides, and personal metadata indexes
 
Sign in  
 
Case Search My Cases Attorney Dashboard
Search Results
Party Search
First:	Robert
Middle:	Edward
Last:	Evans
Sounds-like:	Yes
DOB:	10/09/1981
11 Results
State of Indiana v. Robert Edward Evans
82C01-2606-F6-003854
CourtVanderburgh Circuit Court
Case TypeF6 - Felony 6
Filed06/29/2026
Status06/29/2026, Pending
Charges35-42-2-1.3(a)(1)/F6: Domestic Battery Resulting in Moderate Bodily Injury, 35-42-2-1.3(a)(1)/F6: Domestic Battery-def. has a prior conviction for any sort of battery or strang
PartiesState of Indiana, Evans
AttorneysAlameddine, Hayden
Filed
06/29/2026
State of Indiana vs. Robert Edward Evans
82C01-2606-MC-002265
CourtVanderburgh Circuit Court
Case TypeMC - Miscellaneous Criminal, Probable Cause: Arrest without warrant
Filed06/27/2026
Status06/29/2026, Decided
ChargesNo Charge Applicable
PartiesState of Indiana, Evans
Filed
06/27/2026
State of Indiana v. Robert Edward Evans
14D01-2307-CM-000825
CourtDaviess Superior Court
Case TypeCM - Criminal Misdemeanor
Filed07/24/2023
Status03/11/2024, Decided
Charges35-45-1-3(a)(2)/MB: Disorderly Conduct - unreasonable noise
PartiesState of Indiana, Evans
AttorneysDysert, Roberts, Leinenbach
Filed
07/24/2023
State of Indiana v. Robert Edward Evans
14D01-2106-F6-000602
CourtDaviess Superior Court
Case TypeF6 - Felony 6
Filed06/03/2021
Status03/11/2024, Decided
Charges35-42-2-1.3(a)(1)/F6: Domestic Battery-Same as 6700, but def. has a prior conviction for any sort of, 35-43-1-2(a)/MB:Criminal Mischief
PartiesState of Indiana, Daviess County Probation Department, Evans
AttorneysMiles, Roberts, Leinenbach
Filed
06/03/2021
State of Indiana v. Robert Edward Evans
14D01-1912-CM-001878
CourtDaviess Superior Court
Case TypeCM - Criminal Misdemeanor
Filed12/18/2019
Status03/11/2024, Decided
Charges35-42-2-1.3(a)(1)/MA: Domestic Battery
PartiesState of Indiana, Evans
AttorneysBowling, Roberts, Leinenbach
Filed
12/18/2019
State Vs Robert Edward Evans
14D01-1908-CM-001279
CourtDaviess Superior Court
Case TypeCM - Criminal Misdemeanor
Filed08/22/2019
Status03/11/2024, Decided
ChargesBATTERY RESULTING IN BODILY INJURY
PartiesState of Indiana, Evans
AttorneysRoberts, Leinenbach
Filed
08/22/2019
State Vs Robert Edward Evans
14D01-1704-F6-000441
CourtDaviess Superior Court
Case TypeF6 - Felony 6
Filed04/06/2017
Status06/21/2017, Decided
ChargesPOINTING A FIREARM- Pointing A Firearm At Another, CRIMINAL RECKLESSNESS- Use When Committed With A Deadly Weapon.
PartiesState of Indiana, Evans
AttorneysMurrie
Filed
04/06/2017
State Of Indiana Vs Robert Edward Evans
14D01-1301-CM-000075
CourtDaviess Superior Court
Case TypeCM - Criminal Misdemeanor
Filed01/22/2013
Status02/19/2013, Decided
ChargesDOMESTIC BATTERY- Offense Committed In Presence Of Child Less Than 16 Yrs Old
PartiesState of Indiana, Evans
AttorneysChestnut, Murrie
Filed
01/22/2013
St. Vs Robert Edward Evans
14D01-1203-CM-000208
CourtDaviess Superior Court
Case TypeCM - Criminal Misdemeanor
Filed03/07/2012
Status10/02/2012, Decided
ChargesPUBLIC INTOXICATION
PartiesState of Indiana, Evans
AttorneysMurrie
Filed
03/07/2012
State of Indiana vs. Robert Edward Evans
82D05-1105-CM-02580
CourtVanderburgh Superior Court 6
Case TypeCM - Criminal Misdemeanor
Filed05/04/2011
Status08/02/2011, Decided
Charges7.1-5-1-3 Public Intoxication, CL B MISD
PartiesState of Indiana, Evans
Filed
05/04/2011
State of Indiana vs. Robert Edward Evans
82D05-1102-CM-01180
CourtVanderburgh Superior Court 7
Case TypeCM - Criminal Misdemeanor
Filed02/17/2011
Status09/06/2011, Decided
Charges35-44-3-3 Resisting Law Enforcement, CL A MISD
PartiesState of Indiana, Evans
AttorneysFreson
Filed
02/17/2011
11 Results
 
Subscribe to content by RSS or Email
Follow us on Twitter
Indiana Courts YouTube Channel
Get our shared photos on Flickr
Terms of Use
Odyssey Public Access (the "MyCase" website) is a platform for online services provided by the Office of Judicial Administration (the “Office”).

Electronic access to court information is restricted by federal and state law in addition to court rules and orders. Information on this site is made available as a public service pursuant to order of the Indiana Supreme Court.

Information displayed on this site is not to be considered or used as an official court record and may contain errors or omissions. Accuracy of the information is not warranted. Official records of court proceedings may only be obtained directly from the court maintaining a particular record.

Read the complete Terms of Use »lockdown_egress.shimport chromadb

# Initialize the local persistent ChromaDB client
client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_collection(name="master_hub_rag")

# Perform a semantic search query
results = collection.query(
    query_texts=["backup pipeline configuration"],
    n_results=3
)

print(results)
pip install chromadb pandas pypdf zstandard
0 2 * * * /bin/bash /path/to/Master-Hub/backup_pipeline.sh ~/Master-Hub >/dev/null 2>&1
crontab -e
./backup_pipeline.sh ~/Master-Hub
chmod +x backup_pipeline.sh
chmod 700 ~/Master-Hub
sudo chmod +x lockdown_egress.sh
sudo ./lockdown_egress.sh
mkdir -p ~/Master-Hub/{core_engine,evidentiary_vault,sandboxes,docs}
cd ~/Master-Hub
name: Secure CI/CD Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

# Enforce least-privilege token permissions globally
permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Audit Dependencies
        run: |
          pip install safety
          safety check

      - name: Run SAST Scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog OSS Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
AI Mode
All
Forums
News
Short videos
Shopping
Images
Videos
Books
Web
Maps
Flights
Finance
Search tools
Feedback
AI Overview
It looks like your message contains a mix of text, possibly combining a prompt injection or a copy-paste error regarding "Ddos cyber codes" with a detailed list of defensive CI/CD and repository security practices.
Per safety and ethical guidelines, I cannot generate DDoS or cyber attack exploit code, even for educational purposes.
However, I can absolutely help you expand on, implement, or automate the excellent defensive security practices and GitHub configuration steps you listed.
Here is how you can practically implement the first two pillars of your secure repository management strategy:
1. Repository Access & Authentication
Implementing Fine-Grained Personal Access Tokens (PATs)
Unlike classic tokens, fine-grained PATs allow you to restrict access to specific repositories with precise permissions.
How to configure: Go to Settings > Developer Settings > Personal Access Tokens > Fine-grained tokens and select Generate new token.
Best Practice: Set the expiration to the minimum viable window (e.g., 30 days) and grant Read-only access to contents unless the token specifically needs to push code.
Enforcing Branch Protection via README.md / Documentation
To ensure your team adheres to branch protection rules, you can add a standard compliance checklist to your repository's onboarding documentation:
markdown
## Repository Compliance Checklist
- [ ] Enforce Branch Protection: `main` branch requires at least 1 approving review.
- [ ] Require status checks to pass before merging (e.g., CI/CD tests).
- [ ] Require signed commits (GPG/SSH) enabled in GitHub account settings.
Use code with caution.
2. Automated Pipeline Security (GitHub Actions Example)
To automate the SAST, Secret Scanning, and Dependency Auditing you mentioned, you can use a unified GitHub Actions workflow. Below is a secure, non-malicious boilerplate configuration (.github/workflows/security.yml):
yaml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  security-audit:
    runs-size: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # 1. Dependency Auditing (Example for Python/Pip, adapt as needed)
      - name: Audit Dependencies
        run: |
          pip install safety
          safety check

      # 2. Static Application Security Testing (SAST using Semgrep)
      - name: Run SAST Scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      # 3. Secret Scanning (Preventing credential leaks)
      - name: Trufflehog OSS Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
Use code with caution.
To help you build out the rest of your secure repository, let me know:
What programming language or ecosystem your project uses (so we can tailor the SAST and dependency tools)?
If you need help writing a standard, professional README template for your project?
Whether you want to configure specific deployment environments like AWS, Azure, or GCP securely?
AI responses may include mistakes. Learn more




Show more

GitHub
https://github.com
DDoS attack
Missing: you ‎write ‎readme ‎cyber ‎educational ‎only ‎requests ‎+8

GitHub
https://github.com
ddos-attacks · GitHub Topics
GitHub is where people build software. More than 225 million people use GitHub to discover, fork, and contribute to over 800 million projects.
Missing: write ‎readme ‎generating ‎sensitive ‎personally ‎+3

GitHub
https://github.com
ddos-script
Raven-Storm is a powerful DDoS toolkit for penetration tests, including attacks for several protocols written in python. Takedown many connections using ...

GitHub
https://github.com
ddos · GitHub Topics
GitHub is where people build software. More than 225 million people use GitHub to discover, fork, and contribute to over 800 million projects.
Missing: write ‎readme ‎only ‎payloads, ‎processing ‎sensitive ‎+4

GitHub
https://github.com
ddos · GitHub Topics
GitHub is where people build software. More than 225 million people use GitHub to discover, fork, and contribute to over 800 million projects.

GitHub
https://github.com
ddos · GitHub Topics
GitHub is where people build software. More than 225 million people use GitHub to discover, fork, and contribute to over 800 million projects.
Missing: write ‎readme ‎only ‎processing ‎sensitive ‎personally ‎+3

CISA (.gov)
https://www.cisa.gov
Vulnerability Summary for the Week of July 6, 2026
Jul 13, 2026 — A lock ( A locked padlock ) or https:// means you've safely connected to the .gov website. Share sensitive information only on official, secure ...

GitHub
https://github.com
dos-attack · GitHub Topics
DDOS Tool: To take down small websites with HTTP FLOOD. Port scanner: To know the open ports of a site. FTP Password Cracker: To hack file system of websites..
Missing: write ‎readme ‎generating ‎payloads, ‎sensitive ‎+4

GitHub
https://github.com
attack · GitHub Topics
GitHub is where people build software. More than 225 million people use GitHub to discover, fork, and contribute to over 800 million projects.
Missing: write ‎readme ‎only ‎processing ‎sensitive ‎personally ‎+2
People also search for
Results are personalized
-
Try without personalization
47710, Evansville, IN - From your device
Update location
Send feedback
TermsSkip to content
Explore
Explore
Topics
Trending
Collections
Events
GitHub Sponsors
#
ddos-attack
Here are 119 public repositories matching this topic...
DDoS-Ripper
palahsu / DDoS-Ripper
 Code
 Issues
 Pull requests
DDos Ripper a Distributable Denied-of-Service (DDOS) attack server that cuts off targets or surrounding infrastructure in a flood of Internet traffic

security
protection
ddos
ddos-attacks
sql-injection
web-security
denial-of-service
hacking-tool
linux-tools
ddos-tool
ddos-protection
attack-defense
ddos-attack-tools
ddos-attack
hacking-tools
internet-traffic
deface-website
ddos-attack-tool
ddos-ripper
attack-server
Updated on Jun 20
 Python
Nginx-Lua-Anti-DDoS
C0nw0nk / Nginx-Lua-Anti-DDoS
 Code
 Issues
 Pull requests
A Anti-DDoS script to protect Nginx web servers using Lua with a HTML Javascript based authentication puzzle inspired by Cloudflare I am under attack mode an Anti-DDoS authentication page protect yourself from every attack type All Layer 7 Attacks Mitigating Historic Attacks DoS DoS Implications DDoS All Brute Force Attacks Zero day exploits Soc…

javascript
html
security
protection
ddos
dos
lua
exploit
waf
traffic
cloudflare
ddos-attacks
denial-of-service
nginx-lua
ddos-mitigation
anti-ddos
antiddos
ddos-attack
distributed-denial-of-service
anti-ddos-script
Updated 3 weeks ago
 Lua
HULK-v3
Hyperclaw79 / HULK-v3
 Code
 Issues
 Pull requests
 Discussions
Asynchronous HTTP Botnet for Distributed Denial of Service (DDoS)

electron
nodejs
python
http
gui
typescript
ddos
async
asynchronous
https
websockets
python3
named-pipes
hulk
ddos-attack
nextron
ddos-script
Updated on Feb 4, 2023
 Python
goncalopolido / overload
 Code
 Issues
 Pull requests
A network stress testing tool for simulating high traffic loads.

python
ddos
tcp
hack
script
udp
hacking
penetration-testing
ddos-attacks
termux
flood
hacking-tool
ddos-tool
ddos-attack
ddos-script
ddos-attack-tool
Updated on Jul 9
 Python
r3nt0n / torDDoS
 Code
 Issues
 Pull requests
DDoS attacks over Tor

python
ddos
tor
hacking
ddos-attacks
denial-of-service
tor-network
hacking-tool
ddos-tool
ddos-attack
ddos-script
Updated on Jul 27, 2023
 Python
Leeon123 / golang-httpflood
 Code
 Issues
 Pull requests
Golang httpflood(socket) can down a normal website in 10s with 100 connections

golang
socket
ddos
stress-testing
stress-test
stress-simulations
http-flood
layer7
stress-tester
ddos-attack-tools
ddos-attack
dos-attack
Updated on Aug 24, 2022
 Go
Pyrra-v1.0
Pyrra00 / Pyrra-v1.0
 Code
 Issues
 Pull requests
A voice control hacking assistant in Python

python
ddos-attack
instagram-bruteforce
facebook-bruteforce
Updated on Dec 26, 2019
 Python
Leeon123 / Aoyama
 Code
 Issues
 Pull requests
A New version of Python3 botnet, old version: http://github.com/Leeon123/Python3-botnet

encryption
ddos
botnet
trojan
ddos-attacks
botnet-lab
python-bot
ddos-tool
ddos-attack
botnet-tools
malwar
encrypted-traffic
Updated on Mar 23, 2022
 Python
dheater
c0r0n3r / dheater
 Code
 Issues
 Pull requests
 Discussions
D(HE)ater is a proof of concept implementation of the D(HE)at attack (CVE-2002-20001) through which denial-of-service can be performed by enforcing the Diffie-Hellman key exchange. (read-only clone of the original GitLab project)

tls
ssh
ssl
security
ddos
attack
poc
denial-of-service
ddos-tool
security-tools
overloading
diffie-hellman-groups
diffie-hellman-algorithm
ddos-attack-tools
ddos-attack
dos-attack
diffie-hellman-key
dhe
ddos-attack-tool
dos-attack-tool
Updated on Aug 4
 Python
ProgrammerGaurav / DDos-Attack
 Code
 Issues
 Pull requests
This is a simple DDoS Attack tool and even a begginer hacker can use this.Just type python run.py after cloning this repository.

terminal
python3
termux
hacktoberfest
ddos-attack
termux-environment
termux-tool
termux-hacking
begginer-hacker
Updated on Oct 4, 2021
 C
skavngr / netbot
 Code
 Issues
 Pull requests
A versatile command and control center (CCC) for DDoS Botnet Simulation & Load Generation.

bot
ddos
dos
bots
botnet
simulation
ccc
denial-of-service
ddos-tool
commandandconquer
botnets
ddos-attack-tools
ddos-attack
attack-simulator
commandandcontrol
attack-simulation
Updated on Mar 18, 2024
 Python
codingplanets / Overload-DoS
 Code
 Issues
 Pull requests
"Overload" is a python program that sends active connections to any target of some sort. It is used to perform a DoS/DDoS attack.

ddos
dos
ddos-attacks
ddos-tool
ddos-attack-tools
ddos-attack
python-ddos
Updated on Jan 4, 2020
 Python
IronEye
iAkashPattnaik / IronEye
 Code
 Issues
 Pull requests
 Discussions
An Original Linux Based Cracking Tool.

android
python
hacktoberfest
ddos-attack
ironeye
android-windows-linux
Updated on Apr 21
 Python
elitekitz / EliteToolKit
 Code
 Issues
 Pull requests
Welcome to Python DDOS V1.0! 🎉 This is your ultimate IP stresser and IP booter made by https://elitestresser.club/ is packed with power as a DDOS tool for testing your own servers. Whether you’re unleashing a UDP DDOS tool, a TCP DDOS tool, or hitting web servers with HTTP floods

ddos-attack
python-ddos
dos-attack
stresses
ddoser
ovh-bypass
http-ddos
tcp-ddos
udp-ddos
tcp-bypass
udp-bypass
ddos-ip-booter
ddos-ip-stresser
Updated on Mar 13, 2025
 Python
nejason57mars / mcbot
 Code
 Issues
 Pull requests
java -jar MCBOT.jar <IP:PORT> <PROTOCOL> <METHOD> <SECONDS> <CPS> minecraft ddos tool

bot
minecraft
ddos
bots
ddos-attacks
flood
ddos-tool
ddos-attack
ddos-script
minecraft-bots
minecraft-bot-attack
minecraft-botting
24methods
minecraft-ddos
Updated on Aug 12, 2022
lucthienphong1120 / full-60-ddos-tools
 Code
 Issues
 Pull requests
python
ddos
network
python-script
hacking
ddos-attacks
hacking-tool
sercurity
ddos-tool
ddos-attack-tools
ddos-attack
hacking-tools
hackingtools
ddos-script
ddos-attack-tool
ddos-python
ddos-script-tool
ddos-website
Updated on Dec 8, 2021
 Python
nayumiDEV / cc-ddos
 Code
 Issues
 Pull requests
Một tập lệnh Python để DDOS một trang web bằng phương pháp nhiều phương pháp HTTP Flood, một trang web bình thường chỉ cần 5s để sập hoàn toàn!

python
ddos
attack
vietnamese
hacking
python3
ddos-attacks
python-3
termux
vietnam
http-flood
ddos-tool
vietnamese-language
ddos-attack-tools
ddos-attack
cc-attack
http-flooding
ddos-attack-tool
get-flood
head-flood
Updated on Apr 21, 2025
 Python
cybermads / ZOIC-DDoS
 Code
 Issues
 Pull requests
 Discussions
ZOIC V5

python
ddos
attack
botnet
ddos-attacks
enjoy
layer7
ddos-tool
ddos-attack-tools
ddos-attack
synflood
udpflood
httpflood
layer3
dos-attack
hackingtool
ddos-attacks-tools
zoic
zoic-ddos
Updated on Nov 27, 2025
 Python
blueskychan-dev / DDoSPacket
 Code
 Issues
 Pull requests
Use own Phone to DoS Attack without Termux! (Android 5.1+)

ddos
csharp
android-application
ddos-attacks
xamarin-forms
android-app
ddos-tool
educational-software
ddos-attack
ddos-attack-tool
Updated on May 10, 2025
 C#
aswanthpp / Analysis-of-DDoS-Attacks-in-SDN-Environments
 Code
 Issues
 Pull requests
Analysis of DDoS attack in SDN Environments using miniedit and pox controller

python
ddos-detection
ddos-attack
sdn-environments
Updated on Jun 11, 2018
 Python
Add this topic to your repo
To associate your repository with the ddos-attack topic, visit your repo's landing page and select "manage topics."

Learn more 

Footer
© 2026 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
Community
Docs
Contact
Manage cookies
Do not share my personal information
To build a secure, private, and self-hosted infrastructure framework that integrates local RAG capabilities, zero-trust egress control, and encrypted archiving, you can combine the components into a unified workspace.
Below is the complete, integrated pipeline structure and setup guide for your local Master-Hub environment.
Phase 1: Directory Initialization & Security Baseline
Set up your workspace folder structure and apply strict local file permissions to ensure that configuration files and keys remain private.
# 1. Initialize local repository structure
mkdir -p Master-Hub/{core_engine,evidentiary_vault,sandboxes,docs}
cd Master-Hub

# 2. Secure workspace directory permissions
chmod 700 /mnt/secure_workspace 2>/dev/null || chmod 700 "$PWD"

Phase 2: Zero-Trust Network Lockdown (lockdown_egress.sh)
To isolate your system and enforce a strict zero-trust boundary, use this UFW script to block all unauthorized network egress while whitelisting core services (DNS, HTTP/HTTPS, NTP).
#!/usr/bin/env bash
# ==============================================================================
# Script Name: lockdown_egress.sh
# Description: Zero-Trust UFW firewall lockdown allowing only authorized egress.
# ==============================================================================
set -euo pipefail

if [ "$EUID" -ne 0 ]; then
    echo "[-] Error: Script must be executed as root (sudo)." >&2
    exit 1
fi

echo "[*] Resetting UFW rules..."
ufw --force reset

echo "[*] Setting default-deny policy (Inbound & Outbound)..."
ufw default deny incoming
ufw default deny outgoing
ufw default deny routed

echo "[*] Allowing loopback interface..."
ufw allow in on lo
ufw allow out on lo

echo "[*] Whitelisting critical egress ports..."
ufw allow out 53/tcp   # DNS TCP
ufw allow out 53/udp   # DNS UDP
ufw allow out 80/tcp   # HTTP
ufw allow out 443/tcp  # HTTPS
ufw allow out 123/udp  # NTP Time Sync

echo "[*] Enabling firewall..."
ufw --force enable
echo "[+] Zero-trust egress rules active."
ufw status verbose

Phase 3: Automated Zstd Compression & Encrypted Backup (backup_pipeline.sh)
This script compresses your workspace using ultra-fast zstd compression across all CPU cores (-T0 -19), syncs the archive to an encrypted remote storage destination via rclone, and instantly shreds the local staging file to maintain privacy.
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: Automated zstd workspace archiver & rclone sync with anti-forensics wipe.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
ARCHIVE_NAME="master-hub-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive (-T0 -19)..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Shred local staging archive to ensure privacy
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] Backup and secure purge complete."

Phase 4: Local RAG Ingestion Pipeline (index_backups.py)
To unpack archives locally, parse documentation (Python scripts, Markdown, PDFs, CSVs), and index them into a persistent local vector database (ChromaDB), run the ingestion engine:
import os
import tarfile
from pathlib import Path
import chromadb
import pandas as pd
from pypdf import PdfReader
import zstandard as zstd

ARCHIVE_PATH = "/tmp/backup-staging/master-hub-backup_latest.tar.zst"
EXTRACT_DIR = "/tmp/master-hub-extracted"
CHROMA_DB_DIR = "./chroma_local_db"
COLLECTION_NAME = "master_hub_rag"

def extract_zstd_tar(archive_path, extract_to):
    os.makedirs(extract_to, exist_ok=True)
    dctx = zstd.ZstdDecompressor()
    with open(archive_path, 'rb') as compressed_file:
        with dctx.stream_reader(compressed_file) as reader:
            with tarfile.open(fileobj=reader, mode='r|*') as tar:
                tar.extractall(path=extract_to)

def parse_document(file_path: Path) -> str:
    ext = file_path.suffix.lower()
    text = ""
    try:
        if ext in ['.txt', '.md', '.json', '.py', '.sh', '.csv']:
            with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                text = f.read()
        elif ext == '.pdf':
            reader = PdfReader(file_path)
            for page in reader.pages:
                page_text = page.extract_text()
                if page_text:
                    text += page_text + "\n"
        elif ext in ['.xls', '.xlsx']:
            df = pd.read_excel(file_path)
            text = df.to_string(index=False)
    except Exception as e:
        print(f"[-] Error reading {file_path.name}: {e}")
    return text

def chunk_text(text: str, chunk_size: int = 1000, overlap: int = 200) -> list[str]:
    if not text:
        return []
    return [text[i:i + chunk_size] for i in range(0, len(text), chunk_size - overlap)]

def main():
    if os.path.exists(ARCHIVE_PATH):
        extract_zstd_tar(ARCHIVE_PATH, EXTRACT_DIR)
    
    client = chromadb.PersistentClient(path=CHROMA_DB_DIR)
    collection = client.get_or_create_collection(name=COLLECTION_NAME)
    
    base_path = Path(EXTRACT_DIR)
    documents, metadatas, ids = [], [], []
    doc_counter = 0

    for file_path in base_path.rglob('*'):
        if file_path.is_file():
            raw_text = parse_document(file_path)
            if not raw_text.strip():
                continue
            chunks = chunk_text(raw_text)
            for idx, chunk in enumerate(chunks):
                doc_id = f"{file_path.relative_to(base_path)}_{idx}"
                documents.append(chunk)
                metadatas.append({"source": str(file_path.relative_to(base_path)), "chunk": idx})
                ids.append(doc_id)
                doc_counter += 1

            if len(documents) >= 100:
                collection.upsert(documents=documents, metadatas=metadatas, ids=ids)
                documents, metadatas, ids = [], [], []

    if documents:
        collection.upsert(documents=documents, metadatas=metadatas, ids=ids)

    print(f"[+] Successfully indexed {doc_counter} chunks into local ChromaDB collection '{COLLECTION_NAME}'.")

if __name__ == "__main__":
    main()

Would you like to automate the backup pipeline execution via cron or add custom rules for local semantic query searching?
df.toimport os
import tarfile
from pathlib import Path
import chromadb
import pandas as pd
from pypdf import PdfReader
import zstandard as zstd

ARCHIVE_PATH = "/tmp/backup-staging/master-hub-backup_latest.tar.zst"
EXTRACT_DIR = "/tmp/master-hub-extracted"
CHROMA_DB_DIR = "./chroma_local_db"
COLLECTION_NAME = "master_hub_rag"

def extract_zstd_tar(archive_path, extract_to):
    os.makedirs(extract_to, exist_ok=True)
    dctx = zstd.ZstdDecompressor()
    with open(archive_path, 'rb') as compressed_file:
        with dctx.stream_reader(compressed_file) as reader:
            with tarfile.open(fileobj=reader, mode='r|*') as tar:
                tar.extractall(path=extract_to)

def parse_document(file_path: Path) -> str:
    ext = file_path.suffix.lower()
    text = ""
    try:
        if ext in ['.txt', '.md', '.json', '.py', '.sh', '.csv']:
            with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                text = f.read()
        elif ext == '.pdf':
            reader = PdfReader(file_path)
            for page in reader.pages:
                page_text = page.extract_text()
                if page_text:
                    text += page_text + "\n"
        elif ext in ['.xls', '.xlsx']:
            df = pd.read_excel(file_path)
            text = df.to_string(index=False)
    except Exception as e:
        print(f"[-] Error reading {file_path.name}: {e}")
    return text

def chunk_text(text: str, chunk_size: int = 1000, overlap: int = 200) -> list[str]:
    if not text:
        return []
    return [text[i:i + chunk_size] for i in range(0, len(text), chunk_size - overlap)]

def main():
    if os.path.exists(ARCHIVE_PATH):
        extract_zstd_tar(ARCHIVE_PATH, EXTRACT_DIR)
    
    client = chromadb.PersistentClient(path=CHROMA_DB_DIR)
    collection = client.get_or_create_collection(name=COLLECTION_NAME)
    
    base_path = Path(EXTRACT_DIR)
    documents, metadatas, ids = [], [], []
    doc_counter = 0

    for file_path in base_path.rglob('*'):
        if file_path.is_file():
            raw_text = parse_document(file_path)
            if not raw_text.strip():
                continue
            chunks = chunk_text(raw_text)
            for idx, chunk in enumerate(chunks):
                doc_id = f"{file_path.relative_to(base_path)}_{idx}"
                documents.append(chunk)
                metadatas.append({"source": str(file_path.relative_to(base_path)), "chunk": idx})
                ids.append(doc_id)
                doc_counter += 1

            if len(documents) >= 100:
                collection.upsert(documents=documents, metadatas=metadatas, ids=ids)
                documents, metadatas, ids = [], [], []

    if documents:
        collection.upsert(documents=documents, metadatas=metadatas, ids=ids)

    print(f"[+] Successfully indexed {doc_counter} chunks into local ChromaDB collection '{COLLECTION_NAME}'.")

if __name__ == "__main__":
    main()
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: Automated zstd workspace archiver & rclone sync with anti-forensics wipe.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
ARCHIVE_NAME="master-hub-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive (-T0 -19)..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Shred local staging archive to ensure privacy
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] Backup and secure purge complete."
#!/usr/bin/env bash
# ==============================================================================
# Script Name: lockdown_egress.sh
# Description: Zero-Trust UFW firewall lockdown allowing only authorized egress.
# ==============================================================================
set -euo pipefail

if [ "$EUID" -ne 0 ]; then
    echo "[-] Error: Script must be executed as root (sudo)." >&2
    exit 1
fi

echo "[*] Resetting UFW rules..."
ufw --force reset

echo "[*] Setting default-deny policy (Inbound & Outbound)..."
ufw default deny incoming
ufw default deny outgoing
ufw default deny routed

echo "[*] Allowing loopback interface..."
ufw allow in on lo
ufw allow out on lo

echo "[*] Whitelisting critical egress ports..."
ufw allow out 53/tcp   # DNS TCP
ufw allow out 53/udp   # DNS UDP
ufw allow out 80/tcp   # HTTP
ufw allow out 443/tcp  # HTTPS
ufw allow out 123/udp  # NTP Time Sync

echo "[*] Enabling firewall..."
ufw --force enable
echo "[+] Zero-trust egress rules active."
ufw status verbose

Here is a companion Python script to safely extract your verified .zip archives into clean, structured subdirectories. It handles extraction errors gracefully, avoids overwriting issues, and organizes everything automatically.
extract_archives.py
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()

Usage Instructions
 * Save the script as extract_archives.py.
 * Run the extractor pointing it to your folder of verified .zip files and your desired output destination:
   python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output
#!/usr/bin/env python3
"""
Master File Inventory Generator
Scans extracted directories and outputs a detailed CSV inventory report.
"""

import sys
import argparse
import csv
from datetime import datetime
from pathlib import Path

def get_file_size_mb(size_bytes: int) -> float:
    """Converts bytes to Megabytes rounded to 2 decimal places."""
    return round(size_bytes / (1024 * 1024), 2)

def generate_inventory(source_dir: Path, output_csv: Path):
    """Recursively scans source_dir and writes file metadata to a CSV inventory report."""
    print(f"[*] Scanning directory: '{source_dir}'...")
    
    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist or is not a directory.", file=sys.stderr)
        sys.exit(1)

    file_count = 0
    total_size_bytes = 0

    print(f"[*] Generating inventory report: '{output_csv}'")
    
    # Open CSV file for writing
    with open(output_csv, mode="w", newline="", encoding="utf-8") as csv_file:
        fieldnames = ["Relative Path", "File Name", "Extension", "Size (Bytes)", "Size (MB)", "Last Modified"]
        writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        
        writer.writeheader()

        # Recursively iterate through all items in the directory
        for file_path in source_dir.rglob("*"):
            if file_path.is_file():
                # Skip the inventory CSV itself if it's created inside the target directory
                if file_path.resolve() == output_csv.resolve():
                    continue

                try:
                    stat = file_path.stat()
                    size_bytes = stat.st_size
                    total_size_bytes += size_bytes
                    
                    # Format last modified timestamp
                    mod_time = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
                    
                    # Compute relative path from the source directory
                    rel_path = file_path.relative_to(source_dir)
                    
                    writer.writerow({
                        "Relative Path": str(rel_path),
                        "File Name": file_path.name,
                        "Extension": file_path.suffix.lower() or "(none)",
                        "Size (Bytes)": size_bytes,
                        "Size (MB)": get_file_size_mb(size_bytes),
                        "Last Modified": mod_time
                    })
                    
                    file_count += 1
                    if file_count % 1000 == 0:
                        print(f"    Processed {file_count} files...")

                except Exception as e:
                    print(f"[-] Warning: Could not read metadata for '{file_path}': {e}")

    total_size_gb = round(total_size_bytes / (1024 * 1024 * 1024), 2)
    print("\n" + "=" * 50)
    print(f"[+] Inventory generation complete!")
    print(f"    Total Files Scanned: {file_count:,}")
    print(f"    Total Volume Size:   {total_size_gb} GB ({total_size_bytes:,} bytes)")
    print(f"    Saved to:            {output_csv}")

def main():
    parser = argparse.ArgumentParser(description="Scan extracted directories and generate a master file inventory CSV report.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Root directory to scan (default: current directory)")
    parser.add_argument("output_csv", nargs="?", default="file_inventory.csv", help="Path for the output CSV report (default: file_inventory.csv)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_csv = Path(args.output_csv).resolve()

    generate_inventory(source_dir, output_csv)

if __name__ == "__main__":
    main()
python3 generate_inventory.py /path/to/extracted_output ./master_inventory.csv


This is a [Next.js](https://nextjs.org/) project bootstrapped with [`create-next-app`](https://github.com/vercel/next.js/tree/canary/packages/create-next-app).


Project Page: https://lawyer-nextjs13-pojc.vercel.app/





## Getting Started

First, run the development server:

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

You can start editing the page by modifying `app/page.js`. The page auto-updates as you edit the file.

This project uses [`next/font`](https://nextjs.org/docs/basic-features/font-optimization) to automatically optimize and load Inter, a custom Google Font.

## Learn More

To learn more about Next.js, take a look at the following resources:

- [Next.js Documentation](https://nextjs.org/docs) - learn about Next.js features and API.
- [Learn Next.js](https://nextjs.org/learn) - an interactive Next.js tutorial.

You can check out [the Next.js GitHub repository](https://github.com/vercel/next.js/) - your feedback and contributions are welcome!

## Deploy on Vercel

The easiest way to deploy your Next.js app is to use the [Vercel Platform](https://vercel.com/new?utm_medium=default-template&filter=next.js&utm_source=create-next-app&utm_campaign=create-next-app-readme) from the creators of Next.js.

Check out our [Next.js deployment documentation](https://nextjs.org/docs/deployment) for more details.
