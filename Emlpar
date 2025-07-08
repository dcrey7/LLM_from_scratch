import email
import os
import json
import base64
from email.header import decode_header
from email.utils import parsedate_to_datetime
import mimetypes
from pathlib import Path
import tempfile

class EMLParser:
def **init**(self, main_eml_path):
self.main_eml_path = main_eml_path
self.output_folder = None
self.email_counter = 1
self.filename_counter = {}

```
def decode_header_value(self, header_value):
    """Decode email header values that might be encoded"""
    if header_value is None:
        return ""
    
    decoded_parts = decode_header(header_value)
    decoded_string = ""
    
    for part, encoding in decoded_parts:
        if isinstance(part, bytes):
            try:
                if encoding:
                    decoded_string += part.decode(encoding)
                else:
                    decoded_string += part.decode('utf-8', errors='ignore')
            except (UnicodeDecodeError, LookupError):
                decoded_string += part.decode('utf-8', errors='ignore')
        else:
            decoded_string += str(part)
    
    return decoded_string

def get_unique_filename(self, filename):
    """Generate unique filename to avoid conflicts"""
    if not filename:
        filename = "unnamed_attachment"
    
    # Clean the filename
    filename = "".join(c for c in filename if c.isalnum() or c in "._- ")
    
    if filename not in self.filename_counter:
        self.filename_counter[filename] = 0
        return filename
    else:
        self.filename_counter[filename] += 1
        name, ext = os.path.splitext(filename)
        return f"{name}_{self.filename_counter[filename]}{ext}"

def extract_metadata(self, msg):
    """Extract only the most important email metadata"""
    print("🔍 Extracting essential metadata...")
    
    metadata = {}
    
    # Only extract the core essential headers
    essential_headers = {
        'From': 'from',
        'To': 'to', 
        'Cc': 'cc',
        'Bcc': 'bcc',
        'Subject': 'subject',
        'Date': 'date'
    }
    
    for header, key in essential_headers.items():
        value = msg.get(header)
        if value:
            decoded_value = self.decode_header_value(value)
            metadata[key] = decoded_value
            print(f"  ✓ {key}: {decoded_value[:60]}{'...' if len(decoded_value) > 60 else ''}")
        else:
            print(f"  - {key}: Not found")
    
    # Parse and format date properly
    if 'date' in metadata:
        try:
            date_obj = parsedate_to_datetime(msg.get('Date'))
            metadata['readable_date'] = date_obj.strftime('%Y-%m-%d %H:%M:%S')
            print(f"  ✓ readable_date: {metadata['readable_date']}")
        except Exception as e:
            print(f"  ⚠️ Warning: Could not parse date: {e}")
    
    # Calculate email size
    try:
        email_str = str(msg)
        metadata['size_bytes'] = len(email_str.encode('utf-8'))
        metadata['size_kb'] = round(metadata['size_bytes'] / 1024, 2)
        metadata['size_mb'] = round(metadata['size_bytes'] / (1024*1024), 2)
        print(f"  ✓ size: {metadata['size_kb']} KB ({metadata['size_mb']} MB)")
    except Exception as e:
        print(f"  ⚠️ Warning: Could not calculate size: {e}")
    
    print(f"✅ Essential metadata extracted - {len(metadata)} core fields")
    return metadata

def extract_content(self, msg):
    """Extract email content (text/html)"""
    print("📄 Extracting content...")
    
    content = {
        'text': '',
        'html': '',
        'other_parts': []
    }
    
    def extract_part_content(part):
        content_type = part.get_content_type()
        charset = part.get_content_charset() or 'utf-8'
        
        try:
            if content_type == 'text/plain':
                payload = part.get_payload(decode=True)
                if payload:
                    text_content = payload.decode(charset, errors='ignore')
                    content['text'] += text_content
                    print(f"  ✓ Found text content: {len(text_content)} characters")
            
            elif content_type == 'text/html':
                payload = part.get_payload(decode=True)
                if payload:
                    html_content = payload.decode(charset, errors='ignore')
                    content['html'] += html_content
                    print(f"  ✓ Found HTML content: {len(html_content)} characters")
            
            else:
                # Handle other content types
                payload = part.get_payload(decode=True)
                if payload and len(payload) < 50000:  # Only for smaller content
                    try:
                        decoded_content = payload.decode(charset, errors='ignore')
                        content['other_parts'].append({
                            'content_type': content_type,
                            'content': decoded_content
                        })
                        print(f"  ✓ Found other content type: {content_type}")
                    except:
                        # If it's binary data, just note it
                        content['other_parts'].append({
                            'content_type': content_type,
                            'note': 'Binary content (not decoded)'
                        })
                        print(f"  ✓ Found binary content: {content_type}")
        
        except Exception as e:
            print(f"  ⚠️ Warning: Could not extract content from part {content_type}: {e}")
    
    if msg.is_multipart():
        part_count = 0
        for part in msg.walk():
            if not part.is_multipart():
                disposition = part.get('Content-Disposition', '')
                if 'attachment' not in disposition:
                    part_count += 1
                    extract_part_content(part)
        print(f"  📊 Processed {part_count} content parts")
    else:
        extract_part_content(msg)
        print("  📊 Processed single part email")
    
    print(f"✅ Content extraction complete")
    return content

def process_eml_content(self, eml_data, level=0):
    """Process a single EML file content"""
    print(f"\n{'='*50}")
    print(f"🔄 PROCESSING EMAIL AT LEVEL {level}")
    print(f"{'='*50}")
    
    try:
        # Parse the email
        print("📧 Parsing email message...")
        msg = email.message_from_bytes(eml_data)
        print("✅ Email parsed successfully")
        
        # Extract metadata and content
        metadata = self.extract_metadata(msg)
        content = self.extract_content(msg)
        
        # Create the JSON structure
        email_data = {
            'level': level,
            'metadata': metadata,
            'content': content,
            'attachments_info': []
        }
        
        # Process attachments
        print("\n📎 Processing attachments...")
        nested_eml_files = []
        attachment_count = 0
        
        if msg.is_multipart():
            for part in msg.walk():
                if part.get('Content-Disposition') is None:
                    continue
                
                disposition = part.get('Content-Disposition', '')
                if 'attachment' in disposition or part.get_filename():
                    attachment_count += 1
                    
                    filename = part.get_filename()
                    if filename:
                        filename = self.decode_header_value(filename)
                    
                    content_type = part.get_content_type()
                    print(f"  📎 Processing attachment {attachment_count}: {filename}")
                    
                    try:
                        # Get attachment data
                        attachment_data = part.get_payload(decode=True)
                        if attachment_data:
                            
                            # Generate unique filename
                            safe_filename = self.get_unique_filename(filename)
                            
                            # Save attachment to folder
                            attachment_path = os.path.join(self.output_folder, safe_filename)
                            with open(attachment_path, 'wb') as f:
                                f.write(attachment_data)
                            
                            print(f"    ✅ Saved as: {safe_filename} ({len(attachment_data)} bytes)")
                            
                            # Add to email data
                            attachment_info = {
                                'original_filename': filename,
                                'saved_filename': safe_filename,
                                'content_type': content_type,
                                'size_bytes': len(attachment_data),
                                'size_kb': round(len(attachment_data) / 1024, 2)
                            }
                            email_data['attachments_info'].append(attachment_info)
                            
                            # Check if this is a nested EML file
                            if (filename and filename.lower().endswith('.eml')) or content_type == 'message/rfc822':
                                nested_eml_files.append(attachment_data)
                                print(f"    🔄 FOUND NESTED EML - will process recursively!")
                    
                    except Exception as e:
                        print(f"    ❌ Error processing attachment {filename}: {e}")
                        continue
        
        print(f"📊 Total attachments processed: {attachment_count}")
        print(f"🔄 Nested EML files found: {len(nested_eml_files)}")
        
        # Save this email's JSON
        json_filename = f"email.json" if level == 0 else f"email{level + 1}.json"
        json_path = os.path.join(self.output_folder, json_filename)
        
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(email_data, f, indent=2, ensure_ascii=False)
        
        print(f"💾 Saved email data to: {json_filename}")
        
        # Process nested EML files recursively
        if nested_eml_files:
            print(f"\n🔄 Starting recursive processing of {len(nested_eml_files)} nested EML files...")
            for i, nested_eml_data in enumerate(nested_eml_files, 1):
                try:
                    print(f"\n➡️  Processing nested EML {i}/{len(nested_eml_files)}")
                    self.process_eml_content(nested_eml_data, level + 1)
                except Exception as e:
                    print(f"❌ Error processing nested EML {i} at level {level + 1}: {e}")
                    continue
            print(f"✅ Completed recursive processing for level {level}")
        else:
            print("ℹ️  No nested EML files to process")
    
    except Exception as e:
        print(f"❌ Error processing EML at level {level}: {e}")
        raise

def parse(self):
    """Main parsing function"""
    print("🚀 STARTING EML PARSER")
    print("="*60)
    
    try:
        # Create output folder based on main EML filename
        main_filename = os.path.splitext(os.path.basename(self.main_eml_path))[0]
        self.output_folder = os.path.join(os.path.dirname(self.main_eml_path), main_filename)
        
        # Create folder if it doesn't exist
        os.makedirs(self.output_folder, exist_ok=True)
        print(f"📁 Created output folder: {self.output_folder}")
        
        # Read the main EML file
        print(f"📖 Reading main EML file: {self.main_eml_path}")
        with open(self.main_eml_path, 'rb') as f:
            eml_data = f.read()
        
        file_size_kb = round(len(eml_data) / 1024, 2)
        print(f"📊 File size: {file_size_kb} KB")
        
        # Start recursive processing
        print(f"🔄 Starting recursive processing...")
        self.process_eml_content(eml_data, level=0)
        
        print(f"\n" + "="*60)
        print(f"🎉 PROCESSING COMPLETE!")
        print(f"📁 Output folder: {self.output_folder}")
        print(f"📧 Check the folder for all email JSON files and attachments")
        print(f"="*60)
        
    except Exception as e:
        print(f"💥 FATAL ERROR during parsing: {e}")
        raise
```

def main():
“”“Example usage”””
print(“EML Recursive Parser”)
print(”=”*40)

```
# Replace with your EML file path
eml_file_path = input("📥 Enter the path to your .eml file: ").strip()

if not os.path.exists(eml_file_path):
    print("❌ File not found!")
    return

if not eml_file_path.lower().endswith('.eml'):
    print("⚠️  Warning: File doesn't have .eml extension")
    proceed = input("Continue anyway? (y/n): ").strip().lower()
    if proceed != 'y':
        return

try:
    parser = EMLParser(eml_file_path)
    parser.parse()
except Exception as e:
    print(f"💥 Failed to process EML file: {e}")
```

if **name** == “**main**”:
main()



--------/-----------------/-------

import email
import os
import json
import base64
from email.header import decode_header
from email.utils import parsedate_to_datetime
import mimetypes
from pathlib import Path
import tempfile

class EMLParser:
def **init**(self, main_eml_path):
self.main_eml_path = main_eml_path
self.output_folder = None
self.email_counter = 1
self.filename_counter = {}

```
def decode_header_value(self, header_value):
    """Decode email header values that might be encoded"""
    if header_value is None:
        return ""
    
    decoded_parts = decode_header(header_value)
    decoded_string = ""
    
    for part, encoding in decoded_parts:
        if isinstance(part, bytes):
            try:
                if encoding:
                    decoded_string += part.decode(encoding)
                else:
                    decoded_string += part.decode('utf-8', errors='ignore')
            except (UnicodeDecodeError, LookupError):
                decoded_string += part.decode('utf-8', errors='ignore')
        else:
            decoded_string += str(part)
    
    return decoded_string

def get_unique_filename(self, filename):
    """Generate unique filename to avoid conflicts"""
    if not filename:
        filename = "unnamed_attachment"
    
    # Clean the filename
    filename = "".join(c for c in filename if c.isalnum() or c in "._- ")
    
    if filename not in self.filename_counter:
        self.filename_counter[filename] = 0
        return filename
    else:
        self.filename_counter[filename] += 1
        name, ext = os.path.splitext(filename)
        return f"{name}_{self.filename_counter[filename]}{ext}"

def extract_metadata(self, msg):
    """Extract email metadata"""
    metadata = {}
    
    # Basic headers
    headers_to_extract = [
        'From', 'To', 'Cc', 'Bcc', 'Subject', 'Date', 
        'Message-ID', 'Reply-To', 'Return-Path', 'X-Mailer',
        'Content-Type', 'MIME-Version'
    ]
    
    for header in headers_to_extract:
        value = msg.get(header)
        if value:
            metadata[header.lower()] = self.decode_header_value(value)
    
    # Parse date properly
    if 'date' in metadata:
        try:
            date_obj = parsedate_to_datetime(msg.get('Date'))
            metadata['parsed_date'] = date_obj.isoformat()
        except Exception as e:
            print(f"Warning: Could not parse date: {e}")
    
    # Add all other headers
    metadata['all_headers'] = {}
    for key, value in msg.items():
        metadata['all_headers'][key] = self.decode_header_value(value)
    
    return metadata

def extract_content(self, msg):
    """Extract email content (text/html)"""
    content = {
        'text': '',
        'html': '',
        'other_parts': []
    }
    
    def extract_part_content(part):
        content_type = part.get_content_type()
        charset = part.get_content_charset() or 'utf-8'
        
        try:
            if content_type == 'text/plain':
                payload = part.get_payload(decode=True)
                if payload:
                    content['text'] += payload.decode(charset, errors='ignore')
            
            elif content_type == 'text/html':
                payload = part.get_payload(decode=True)
                if payload:
                    content['html'] += payload.decode(charset, errors='ignore')
            
            else:
                # Handle other content types
                payload = part.get_payload(decode=True)
                if payload:
                    try:
                        decoded_content = payload.decode(charset, errors='ignore')
                        content['other_parts'].append({
                            'content_type': content_type,
                            'content': decoded_content
                        })
                    except:
                        # If it's binary data, encode as base64
                        content['other_parts'].append({
                            'content_type': content_type,
                            'content_base64': base64.b64encode(payload).decode('utf-8')
                        })
        
        except Exception as e:
            print(f"Warning: Could not extract content from part {content_type}: {e}")
    
    if msg.is_multipart():
        for part in msg.walk():
            if not part.is_multipart():
                disposition = part.get('Content-Disposition', '')
                if 'attachment' not in disposition:
                    extract_part_content(part)
    else:
        extract_part_content(msg)
    
    return content

def process_eml_content(self, eml_data, level=0):
    """Process a single EML file content"""
    try:
        # Parse the email
        msg = email.message_from_bytes(eml_data)
        
        # Extract metadata and content
        metadata = self.extract_metadata(msg)
        content = self.extract_content(msg)
        
        # Create the JSON structure
        email_data = {
            'level': level,
            'metadata': metadata,
            'content': content,
            'attachments_info': []
        }
        
        # Process attachments
        nested_eml_files = []
        
        if msg.is_multipart():
            for part in msg.walk():
                if part.get('Content-Disposition') is None:
                    continue
                
                disposition = part.get('Content-Disposition', '')
                if 'attachment' in disposition or part.get_filename():
                    
                    filename = part.get_filename()
                    if filename:
                        filename = self.decode_header_value(filename)
                    
                    content_type = part.get_content_type()
                    
                    try:
                        # Get attachment data
                        attachment_data = part.get_payload(decode=True)
                        if attachment_data:
                            
                            # Generate unique filename
                            safe_filename = self.get_unique_filename(filename)
                            
                            # Save attachment to folder
                            attachment_path = os.path.join(self.output_folder, safe_filename)
                            with open(attachment_path, 'wb') as f:
                                f.write(attachment_data)
                            
                            # Add to email data
                            attachment_info = {
                                'original_filename': filename,
                                'saved_filename': safe_filename,
                                'content_type': content_type,
                                'size_bytes': len(attachment_data)
                            }
                            email_data['attachments_info'].append(attachment_info)
                            
                            # Check if this is a nested EML file
                            if (filename and filename.lower().endswith('.eml')) or content_type == 'message/rfc822':
                                nested_eml_files.append(attachment_data)
                                print(f"Found nested EML at level {level}: {filename}")
                    
                    except Exception as e:
                        print(f"Error processing attachment {filename}: {e}")
                        continue
        
        # Save this email's JSON
        json_filename = f"email.json" if level == 0 else f"email{level + 1}.json"
        json_path = os.path.join(self.output_folder, json_filename)
        
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(email_data, f, indent=2, ensure_ascii=False)
        
        print(f"Saved email data to {json_filename} (Level {level})")
        
        # Process nested EML files recursively
        for nested_eml_data in nested_eml_files:
            try:
                self.process_eml_content(nested_eml_data, level + 1)
            except Exception as e:
                print(f"Error processing nested EML at level {level + 1}: {e}")
                continue
    
    except Exception as e:
        print(f"Error processing EML at level {level}: {e}")
        raise

def parse(self):
    """Main parsing function"""
    try:
        # Create output folder based on main EML filename
        main_filename = os.path.splitext(os.path.basename(self.main_eml_path))[0]
        self.output_folder = os.path.join(os.path.dirname(self.main_eml_path), main_filename)
        
        # Create folder if it doesn't exist
        os.makedirs(self.output_folder, exist_ok=True)
        print(f"Created output folder: {self.output_folder}")
        
        # Read the main EML file
        with open(self.main_eml_path, 'rb') as f:
            eml_data = f.read()
        
        # Start recursive processing
        print(f"Starting to process: {self.main_eml_path}")
        self.process_eml_content(eml_data, level=0)
        
        print(f"\n✅ Processing complete!")
        print(f"📁 Output folder: {self.output_folder}")
        print(f"📧 All email data and attachments saved")
        
    except Exception as e:
        print(f"❌ Error during parsing: {e}")
        raise
```

def main():
“”“Example usage”””
# Replace with your EML file path
eml_file_path = input(“Enter the path to your .eml file: “).strip()

```
if not os.path.exists(eml_file_path):
    print("❌ File not found!")
    return

if not eml_file_path.lower().endswith('.eml'):
    print("⚠️  Warning: File doesn't have .eml extension")

try:
    parser = EMLParser(eml_file_path)
    parser.parse()
except Exception as e:
    print(f"❌ Failed to process EML file: {e}")
```

if **name** == “**main**”:
main()
