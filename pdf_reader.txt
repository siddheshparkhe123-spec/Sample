"""
HSBC Email Triage — PDF Text Extractor
=======================================
Reads text from:
  - Normal text-based PDFs
  - Compressed (FlateDecode) PDFs
  - PDFs with embedded attachments (PDF-in-PDF)
  - PDFs with embedded files (.eml, .txt, .docx inside PDF)

Requirements:
    pip install PyPDF2 pdfplumber pymupdf

Usage:
    python pdf_reader.py sample.pdf
    python pdf_reader.py                  # reads all PDFs in current folder
"""

import os
import sys
import io
import zipfile
import tempfile


# ── HELPER: try importing libraries ──────────────────────────────────────────
def try_import(name):
    try:
        return __import__(name)
    except ImportError:
        return None


pypdf2    = try_import('PyPDF2')
pdfplumber = try_import('pdfplumber')
fitz      = try_import('fitz')          # PyMuPDF


# ─────────────────────────────────────────────────────────────────────────────
#  METHOD 1 — PyMuPDF (fitz) — Best for text + embedded files
# ─────────────────────────────────────────────────────────────────────────────
def extract_with_pymupdf(pdf_path):
    """
    Uses PyMuPDF (fitz) to extract:
      - Text from all pages
      - Embedded files (attachments) inside the PDF
    """
    if not fitz:
        return None, []

    text_parts    = []
    embedded_docs = []

    try:
        doc = fitz.open(pdf_path)

        # ── Extract text from each page ──
        for page_num in range(len(doc)):
            page = doc[page_num]
            text = page.get_text("text")
            if text.strip():
                text_parts.append(
                    f"--- Page {page_num + 1} ---\n{text.strip()}"
                )

        # ── Extract embedded files ──
        # PDFs can have attachments embedded using /EmbeddedFiles
        embedded_count = doc.embfile_count()
        if embedded_count > 0:
            print(f"  Found {embedded_count} embedded file(s) in PDF")
            for i in range(embedded_count):
                info = doc.embfile_info(i)
                name = info.get('filename', f'embedded_{i}')
                data = doc.embfile_get(i)
                print(f"  → Embedded: {name} ({len(data)} bytes)")
                embedded_docs.append({
                    'name': name,
                    'data': data,
                    'size': len(data)
                })

        # ── Also check for PDF annotations with file attachments ──
        for page_num in range(len(doc)):
            page = doc[page_num]
            for annot in page.annots():
                if annot.type[0] == 17:  # PDF_ANNOT_FILE_ATTACHMENT
                    try:
                        annot_info = annot.info
                        name = annot_info.get('content', f'annot_{page_num}')
                        fs   = annot.file_info()
                        if fs:
                            data = annot.get_file()
                            print(f"  → Annotation attachment: {name}")
                            embedded_docs.append({
                                'name': name,
                                'data': data,
                                'size': len(data)
                            })
                    except Exception:
                        pass

        doc.close()
        return '\n\n'.join(text_parts), embedded_docs

    except Exception as e:
        print(f"  PyMuPDF error: {e}")
        return None, []


# ─────────────────────────────────────────────────────────────────────────────
#  METHOD 2 — pdfplumber — Good for tables + structured text
# ─────────────────────────────────────────────────────────────────────────────
def extract_with_pdfplumber(pdf_path):
    if not pdfplumber:
        return None

    text_parts = []
    try:
        with pdfplumber.open(pdf_path) as pdf:
            for i, page in enumerate(pdf.pages):
                # Extract plain text
                text = page.extract_text()
                if text and text.strip():
                    text_parts.append(
                        f"--- Page {i + 1} ---\n{text.strip()}"
                    )

                # Extract tables if present
                tables = page.extract_tables()
                for t_idx, table in enumerate(tables):
                    rows = []
                    for row in table:
                        clean = [str(cell or '').strip() for cell in row]
                        rows.append(' | '.join(clean))
                    if rows:
                        text_parts.append(
                            f"--- Page {i+1} Table {t_idx+1} ---\n"
                            + '\n'.join(rows)
                        )

        return '\n\n'.join(text_parts) if text_parts else None

    except Exception as e:
        print(f"  pdfplumber error: {e}")
        return None


# ─────────────────────────────────────────────────────────────────────────────
#  METHOD 3 — PyPDF2 — Fallback for simple PDFs
# ─────────────────────────────────────────────────────────────────────────────
def extract_with_pypdf2(pdf_path):
    if not pypdf2:
        return None

    text_parts = []
    try:
        with open(pdf_path, 'rb') as f:
            reader = pypdf2.PdfReader(f)

            # Check for embedded files in PDF catalog
            embedded_docs = []
            try:
                catalog = reader.trailer['/Root']
                if '/Names' in catalog:
                    names = catalog['/Names']
                    if '/EmbeddedFiles' in names:
                        ef = names['/EmbeddedFiles']
                        if '/Names' in ef:
                            ef_names = ef['/Names']
                            for idx in range(0, len(ef_names) - 1, 2):
                                fname = str(ef_names[idx])
                                fobj  = ef_names[idx + 1].get_object()
                                if '/EF' in fobj:
                                    ef_stream = fobj['/EF']['/F'].get_object()
                                    data = ef_stream.get_data()
                                    print(f"  → Embedded (PyPDF2): {fname}")
                                    embedded_docs.append({
                                        'name': fname,
                                        'data': data,
                                        'size': len(data)
                                    })
            except Exception:
                pass

            # Extract text from pages
            for i, page in enumerate(reader.pages):
                try:
                    text = page.extract_text()
                    if text and text.strip():
                        text_parts.append(
                            f"--- Page {i + 1} ---\n{text.strip()}"
                        )
                except Exception:
                    pass

        return '\n\n'.join(text_parts) if text_parts else None

    except Exception as e:
        print(f"  PyPDF2 error: {e}")
        return None


# ─────────────────────────────────────────────────────────────────────────────
#  METHOD 4 — Raw binary extraction (no library needed)
#  Uses built-in Python — FlateDecode decompression with zlib
# ─────────────────────────────────────────────────────────────────────────────
def extract_raw_python(pdf_path):
    """
    Pure Python — no external libraries.
    Decompresses FlateDecode streams and extracts text.
    Same logic as the HTML DecompressionStream approach.
    """
    import zlib
    import re

    text_parts = []

    try:
        with open(pdf_path, 'rb') as f:
            data = f.read()

        pos = 0
        streams_processed = 0

        while pos < len(data) - 20:
            # Find 'stream' keyword
            idx = data.find(b'stream', pos)
            if idx == -1:
                break

            stream_start = idx + 6
            if stream_start < len(data) and data[stream_start] == 13:
                stream_start += 1
            if stream_start < len(data) and data[stream_start] == 10:
                stream_start += 1

            # Find matching 'endstream'
            end_idx = data.find(b'endstream', stream_start)
            if end_idx == -1:
                break

            stream_bytes = data[stream_start:end_idx]

            # Try zlib decompression (FlateDecode)
            try:
                decompressed = zlib.decompress(stream_bytes)
                text = decompressed.decode('utf-8', errors='replace')

                # Skip font/resource streams
                is_resource = any(k in text[:200] for k in [
                    '/BaseFont', '/FontDescriptor',
                    '/Encoding', 'ColorSpace', '/XObject'
                ])

                if not is_resource:
                    # Extract BT...ET text blocks
                    bt_blocks = re.findall(
                        r'BT([\s\S]*?)ET', text
                    )
                    for block in bt_blocks:
                        # Simple text: (text) Tj
                        tj_parts = re.findall(
                            r'\(([^)]*)\)\s*Tj', block
                        )
                        for t in tj_parts:
                            clean = t.replace('\\n', '\n') \
                                     .replace('\\r', '') \
                                     .replace('\\(', '(') \
                                     .replace('\\)', ')').strip()
                            if len(clean) > 1:
                                text_parts.append(clean)

                        # Array text: [...] TJ
                        tj_arr = re.findall(
                            r'\[([^\]]*)\]\s*TJ', block
                        )
                        for arr in tj_arr:
                            sub = re.findall(r'\(([^)]{1,200})\)', arr)
                            for s in sub:
                                if len(s.strip()) > 1:
                                    text_parts.append(s.strip())

                    # Plain readable text
                    if not text_parts:
                        readable = re.sub(r'[^\x20-\x7E\n\t]', ' ', text)
                        readable = re.sub(r'\s{3,}', ' ', readable).strip()
                        has_email = any(k in readable for k in [
                            'Dear', 'From', 'To', 'Subject',
                            'Please', 'Regards', 'SWAP',
                            'Guarantee', '@', 'trade'
                        ])
                        if len(readable) > 80 and has_email:
                            text_parts.append(readable)

                    streams_processed += 1

            except zlib.error:
                # Not compressed — try raw text extraction
                try:
                    raw = stream_bytes.decode('latin-1', errors='replace')
                    bt_blocks = re.findall(r'BT([\s\S]*?)ET', raw)
                    for block in bt_blocks:
                        tj_parts = re.findall(
                            r'\(([^)]{1,200})\)\s*T[jJ]', block
                        )
                        for t in tj_parts:
                            if len(t.strip()) > 1:
                                text_parts.append(t.strip())
                except Exception:
                    pass

            pos = end_idx + 9
            if streams_processed > 80:
                break

        result = ' '.join(text_parts)
        result = re.sub(r'\s{2,}', ' ', result).strip()
        return result if len(result) > 30 else None

    except Exception as e:
        print(f"  Raw extraction error: {e}")
        return None


# ─────────────────────────────────────────────────────────────────────────────
#  EMBEDDED FILE PROCESSOR
# ─────────────────────────────────────────────────────────────────────────────
def process_embedded_file(name, data):
    """
    Process an embedded file found inside a PDF.
    Handles: PDF, TXT, EML, DOCX
    """
    name_lower = name.lower()
    print(f"\n  Processing embedded: {name} ({len(data)} bytes)")

    # ── Embedded PDF ──
    if name_lower.endswith('.pdf'):
        print(f"  → Embedded PDF detected, extracting text...")
        with tempfile.NamedTemporaryFile(
            suffix='.pdf', delete=False
        ) as tmp:
            tmp.write(data)
            tmp_path = tmp.name
        try:
            result = extract_pdf(tmp_path, depth=1)
            return result
        finally:
            os.unlink(tmp_path)

    # ── Embedded TXT or EML ──
    elif name_lower.endswith(('.txt', '.eml', '.msg')):
        try:
            return data.decode('utf-8', errors='replace')
        except Exception:
            return data.decode('latin-1', errors='replace')

    # ── Embedded DOCX ──
    elif name_lower.endswith('.docx'):
        try:
            with zipfile.ZipFile(io.BytesIO(data)) as z:
                if 'word/document.xml' in z.namelist():
                    xml = z.read('word/document.xml').decode(
                        'utf-8', errors='replace'
                    )
                    import re
                    texts = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml)
                    return ' '.join(texts)
        except Exception as e:
            print(f"  DOCX error: {e}")
        return None

    # ── Unknown type — try as text ──
    else:
        try:
            text = data.decode('utf-8', errors='replace')
            if len(text.strip()) > 20:
                return text
        except Exception:
            pass
        return None


# ─────────────────────────────────────────────────────────────────────────────
#  MAIN EXTRACTOR — tries all methods in order
# ─────────────────────────────────────────────────────────────────────────────
def extract_pdf(pdf_path, depth=0):
    """
    Main function — extracts text from a PDF using
    multiple methods in priority order.

    depth: 0 = top-level PDF, 1 = embedded PDF
    """
    indent = "  " * depth
    print(f"{indent}Reading: {os.path.basename(pdf_path)}")

    all_text   = []
    embedded   = []

    # ── Try methods in order ──────────────────────────────────────────────

    # Method 1: PyMuPDF (best — handles embedded files too)
    if fitz:
        print(f"{indent}  Trying PyMuPDF...")
        text, embedded = extract_with_pymupdf(pdf_path)
        if text and len(text.strip()) > 50:
            print(f"{indent}  ✓ PyMuPDF: {len(text)} chars extracted")
            all_text.append(('PyMuPDF', text))

    # Method 2: pdfplumber (good for tables)
    if pdfplumber and not all_text:
        print(f"{indent}  Trying pdfplumber...")
        text = extract_with_pdfplumber(pdf_path)
        if text and len(text.strip()) > 50:
            print(f"{indent}  ✓ pdfplumber: {len(text)} chars extracted")
            all_text.append(('pdfplumber', text))

    # Method 3: PyPDF2
    if pypdf2 and not all_text:
        print(f"{indent}  Trying PyPDF2...")
        text = extract_with_pypdf2(pdf_path)
        if text and len(text.strip()) > 50:
            print(f"{indent}  ✓ PyPDF2: {len(text)} chars extracted")
            all_text.append(('PyPDF2', text))

    # Method 4: Raw Python (no library)
    if not all_text:
        print(f"{indent}  Trying raw Python extraction...")
        text = extract_raw_python(pdf_path)
        if text and len(text.strip()) > 30:
            print(f"{indent}  ✓ Raw Python: {len(text)} chars extracted")
            all_text.append(('Raw Python', text))

    # ── Process embedded files ────────────────────────────────────────────
    if embedded:
        print(f"\n{indent}  Processing {len(embedded)} embedded file(s)...")
        for ef in embedded:
            ef_text = process_embedded_file(ef['name'], ef['data'])
            if ef_text and len(ef_text.strip()) > 20:
                all_text.append((
                    f"Embedded: {ef['name']}", ef_text
                ))

    # ── Build final output ────────────────────────────────────────────────
    if not all_text:
        return None

    # Use the best (first successful) extraction
    # but include embedded file content too
    final_parts = []

    # Main PDF text
    main_method, main_text = all_text[0]
    final_parts.append(
        f"=== Main PDF ({main_method}) ===\n{main_text}"
    )

    # Embedded files text
    for method, text in all_text[1:]:
        final_parts.append(
            f"\n=== {method} ===\n{text}"
        )

    return '\n\n'.join(final_parts)


# ─────────────────────────────────────────────────────────────────────────────
#  MAIN
# ─────────────────────────────────────────────────────────────────────────────
def main():
    # Check available libraries
    print("=" * 55)
    print("HSBC PDF Text Extractor")
    print("=" * 55)
    print(f"PyMuPDF    : {'✓ available' if fitz       else '✗ not installed (pip install pymupdf)'}")
    print(f"pdfplumber : {'✓ available' if pdfplumber else '✗ not installed (pip install pdfplumber)'}")
    print(f"PyPDF2     : {'✓ available' if pypdf2     else '✗ not installed (pip install PyPDF2)'}")
    print(f"Raw Python : ✓ always available (built-in zlib)")
    print("=" * 55)

    # Get PDF files to process
    if len(sys.argv) > 1:
        pdf_files = sys.argv[1:]
    else:
        # Process all PDFs in current directory
        pdf_files = [
            f for f in os.listdir('.')
            if f.lower().endswith('.pdf')
        ]
        if not pdf_files:
            print("\nNo PDF files found.")
            print("Usage: python pdf_reader.py file.pdf")
            return

    print(f"\nProcessing {len(pdf_files)} file(s)...\n")

    for pdf_path in pdf_files:
        if not os.path.exists(pdf_path):
            print(f"✗ File not found: {pdf_path}")
            continue

        print(f"\n{'─'*55}")
        result = extract_pdf(pdf_path)

        if result:
            print(f"\n✓ Extraction successful!\n")
            print("─" * 55)
            print(result[:3000])          # preview first 3000 chars
            if len(result) > 3000:
                print(f"\n... ({len(result) - 3000} more chars)")
            print("─" * 55)

            # Save to .txt file
            out_path = pdf_path.replace('.pdf', '_extracted.txt')
            with open(out_path, 'w', encoding='utf-8') as f:
                f.write(result)
            print(f"\n✓ Saved to: {out_path}")

        else:
            print(f"\n✗ Could not extract text from: {pdf_path}")
            print("  This PDF may be:")
            print("  - Image-based (scanned) — needs OCR")
            print("  - Password protected")
            print("  - Corrupted")

    print(f"\n{'='*55}")
    print("Done.")


if __name__ == '__main__':
    main()
