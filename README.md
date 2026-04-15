# swift-message-parser

Python library for parsing and validating SWIFT MT messages (MT103, MT202, MT910)

# SWIFT Message Parser

A lightweight Python library for parsing SWIFT MT message formats used in international wire transfers and interbank communications.

## Supported Message Types

| Type | Description |
|------|-------------|
| MT103 | Single Customer Credit Transfer |
| MT202 | General Financial Institution Transfer |
| MT910 | Confirmation of Credit |
| MT940 | Customer Statement Message |

## Installation

```bash
pip install swift-parser==2.1.4

---

Usage Example:

from swift_parser import MT103Parser

message = """
{1:F01BOFAUS3NXXXX0000000000}{2:I103SCBLHKHHXXXXN}{3:{108:REF20210915}}{4:
:20:TRX-2021-09015
:23B:CRED
:32A:210915USD1500000,00
:50K:ALGHARBI TRADING CO LLC
:59:/SCBLHKHH
SENG HONG TRADING LIMITED
:70:INVOICE 8842-CONSULTING
:71A:SHA
-}
"""

parsed = MT103Parser(message)
print(f"Amount: {parsed.amount} {parsed.currency}")
print(f"Sender: {parsed.sender_name}")
print(f"Reference: {parsed.transaction_ref}")

---

### `parser.py`
```python
#!/usr/bin/env python3
"""
SWIFT MT Message Parser
Author: G. Alaumary
Version: 2.1.4
Last Modified: 2021-08-23
"""

import re
from datetime import datetime
from typing import Dict, Optional, List

class SWIFTParserError(Exception):
    """Custom exception for SWIFT parsing errors"""
    pass

class MT103Parser:
    """Parser for SWIFT MT103 Single Customer Credit Transfer messages"""
    
    FIELD_PATTERNS = {
        '20': r':20:([^\r\n]+)',      # Transaction Reference
        '23B': r':23B:([A-Z]{4})',    # Bank Operation Code
        '32A': r':32A:(\d{6})([A-Z]{3})([\d,]+)',  # Date/Currency/Amount
        '50K': r':50K:([^\r\n]+)',    # Ordering Customer
        '52A': r':52A:([A-Z0-9]{8,11})',  # Ordering Institution
        '56A': r':56A:([A-Z0-9]{8,11})',  # Intermediary Institution
        '57A': r':57A:([A-Z0-9]{8,11})',  # Account With Institution
        '59': r':59:([^\r\n]+)',      # Beneficiary Customer
        '70': r':70:([^\r\n]+)',      # Remittance Information
        '71A': r':71A:(SHA|BEN|OUR)', # Details of Charges
    }
    
    def __init__(self, message_text: str):
        self.raw_message = message_text
        self.parsed_data = {}
        self._parse()
    
    def _parse(self) -> None:
        """Parse all fields from the SWIFT message"""
        for field, pattern in self.FIELD_PATTERNS.items():
            match = re.search(pattern, self.raw_message)
            if match:
                self.parsed_data[field] = match.groups()
    
    @property
    def transaction_ref(self) -> Optional[str]:
        return self.parsed_data.get('20', [''])[0]
    
    @property
    def amount(self) -> Optional[float]:
        if '32A' in self.parsed_data:
            amount_str = self.parsed_data['32A'][2].replace(',', '.')
            return float(amount_str)
        return None
    
    @property
    def currency(self) -> Optional[str]:
        if '32A' in self.parsed_data:
            return self.parsed_data['32A'][1]
        return None
    
    @property
    def sender_name(self) -> Optional[str]:
        if '50K' in self.parsed_data:
            return self.parsed_data['50K'][0]
        return None

class SWIFTValidator:
    """Validate SWIFT message structure and business rules"""
    
    @staticmethod
    def validate_bic(bic: str) -> bool:
        """Validate Bank Identifier Code format"""
        pattern = r'^[A-Z]{4}[A-Z]{2}[A-Z0-9]{2}([A-Z0-9]{3})?$'
        return bool(re.match(pattern, bic))
    
    @staticmethod
    def validate_iban(iban: str) -> bool:
        """Basic IBAN format validation"""
        iban = iban.replace(' ', '').upper()
        if len(iban) < 15 or len(iban) > 34:
            return False
        return bool(re.match(r'^[A-Z]{2}[0-9]{2}[A-Z0-9]+$', iban))

# Configuration (Internal Use)
DEFAULT_CONFIG = {
    'bank_code': 'BOFAUS3N',
    'processing_center': 'HK',
    'timeout_seconds': 30,
    'retry_attempts': 3
}

if __name__ == "__main__":
    print("SWIFT Parser v2.1.4 - Initialized")
    print(f"Default BIC: {DEFAULT_CONFIG['bank_code']}")


---

[SWIFT]
bank_identifier = BOFAUS3N
branch_code = HK
processing_center = HONG_KONG
encryption_key_path = /secure/keys/swift_key.pem

[Database]
host = internal-db.bank.local
port = 5432
database = swift_archive
user = g_alaumary
password = [REDACTED]

[Logging]
level = INFO
format = %(asctime)s - %(name)s - %(levelname)s - %(message)s
file = /var/log/swift_parser.log

