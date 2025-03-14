# zX12 - X12 EDI Parser in Zig

zX12 is a library for parsing X12 EDI (Electronic Data Interchange) documents, with a specialized focus on healthcare claim formats (837P, 837I, 837D). 
## Features

- Parse standard X12 EDI documents
- Specialized support for healthcare claims (837)
    - Professional claims (837P)
    - Institutional claims (837I)
    - Dental claims (837D)
- Extract structured data from complex EDI files
- Clean JSON output for easy integration with web services
- C-compatible API for use from other languages with Arena allocation
## Installation

```bash
git clone https://github.com/jjw07006/zX12.git
cd zX12
zig build
```

## Usage

### Basic X12 Document Parsing
- Keep in mind the parser does minimal allocations, so the contents of the X12 document needs to outlive the Zig X12 Document structure and Claim837 structure.

```zig
const std = @import("std");
const x12 = @import("parser.zig");

pub fn main() !void {
        var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
        defer arena.deinit();
        const allocator = arena.allocator();

        // Input X12 document
        const data = "ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER      *210101*1200*^*00501*000000001*0*P*:~...";

        // Parse document
        var document = x12.X12Document.init(allocator);
        defer document.deinit();
        try document.parse(data);

        // Access segments and elements
        if (document.getSegment("ST")) |st_segment| {
                std.debug.print("Transaction type: {s}\n", .{st_segment.elements.items[0].value});
        }
}
```

### Parsing 837 Healthcare Claims

```zig
const std = @import("std");
const x12 = @import("parser.zig");
const claim837 = @import("claim837.zig");

pub fn main() !void {
        var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
        defer arena.deinit();
        const allocator = arena.allocator();

        // Input X12 document (837 claim)
        const data = "ISA*00*..."; // (837 claim data)

        // Parse X12 document
        var document = x12.X12Document.init(allocator);
        defer document.deinit();
        try document.parse(data);

        // Parse as 837 healthcare claim
        var claim = claim837.Claim837.init(allocator);
        defer claim.deinit();
        try claim.parse(&document);

        // Access claim data
        std.debug.print("Claim type: {s}\n", .{@tagName(claim.transaction_type)});
        std.debug.print("Sender: {s}\n", .{claim.sender_id});
        std.debug.print("Billing provider: {s}\n", .{claim.billing_provider.last_name});
        
        // Convert to JSON
        const json = try std.json.stringifyAlloc(allocator, claim, .{});
        defer allocator.free(json);
        std.debug.print("JSON: {s}\n", .{json});
}
```

## API Reference

### X12Document

The core parser for X12 documents:

- `init(allocator)` - Initialize a new X12Document
- `parse(data)` - Parse a raw X12 document
- `getSegment(id)` - Get the first segment with the specified ID
- `getSegments(id, allocator)` - Get all segments with the specified ID

### Claim837

Specialized parser for 837 healthcare claims:

- `init(allocator)` - Initialize a new 837 claim parser
- `parse(document)` - Parse an X12 document as an 837 claim
- `transaction_type` - Type of claim (professional, institutional, dental)
- `billing_provider` - Information about the billing provider
- `subscriber_loops` - List of subscriber information, patients, and claims

### Example
```
ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER       *230101*1200*^*00501*000000001*0*P*:~
GS*HC*SENDER*RECEIVER*20230101*1200*1*X*005010X223A2~
ST*837*0001*005010X223A2~
BHT*0019*00*123*20230101*1200*CH~
NM1*41*2*HOSPITAL INC*****46*123456789~
PER*IC*CONTACT*TE*5551234~
NM1*40*2*INSURANCE CO*****46*987654321~
HL*1**20*1~
NM1*85*2*GENERAL HOSPITAL*****XX*1234567890~
N3*555 HOSPITAL DRIVE~
N4*SOMECITY*CA*90001~
REF*EI*987654321~
HL*2*1*22*1~
SBR*P*18*******MB~
NM1*IL*1*PATIENT*JOHN****MI*123456789A~
N3*123 PATIENT ST~
N4*SOMECITY*CA*90001~
DMG*D8*19500501*M~
CLM*4567832*25000.00***11:B:1*Y*A*Y*Y*A::1*Y*::3~
DTP*434*RD8*20221201-20221210~
HI*ABK:I269*ABF:I4891*ABF:E119*ABF:Z9911~
HI*BE:01:::450.00*BE:02:::600.00*BE:30:::120.00~
HI*BH:A1:D8:20221201*BH:A2:D8:20221130*BH:45:D8:20221201~
HI*BI:70:D8:20221125-20221130*BI:71:D8:20221101-20221110~
LX*1~
SV2*0120*HC:99231*15000.00*UN*10***1~
DTP*472*D8*20221201~
LX*2~
SV2*0270*HC:85025*500.00*UN*5***2~
DTP*472*D8*20221202~
LX*3~
SV2*0450*HC:99291*9500.00*UN*1***3~
DTP*472*D8*20221205~
SE*31*0001~
GE*1*1~
IEA*1*000000001~
```

The above 837 claim will yeild the below json

```json
{
    "transaction_type": "institutional",
    "transaction_control_number": "005010X223A2",
    "sender_id": "SENDER",
    "receiver_id": "RECEIVER",
    "subscriber_loops": [
        {
            "hl_id": "2",
            "sbr_payer_responsibility": "P",
            "sbr_individual_relationship": "18",
            "sbr_reference_id": "",
            "sbr_claim_filing_code": "MB",
            "patients": [
                {
                    "entity_type": "person",
                    "last_name": "PATIENT",
                    "first_name": "JOHN",
                    "birth_date": "19500501",
                    "gender": "M"
                }
            ],
            "claims": [
                {
                    "claim_id": "4567832",
                    "total_charges": "25000.00",
                    "place_of_service": "11",
                    "occurrence_span_codes": [
                        {
                            "code": "70",
                            "qualifier": "BI",
                            "from_date": "20221125",
                            "to_date": "20221130"
                        },
                        {
                            "code": "71",
                            "qualifier": "BI",
                            "from_date": "20221101",
                            "to_date": "20221110"
                        }
                    ],
                    "occurrence_codes": [
                        {
                            "code": "A1",
                            "date": "20221201",
                            "qualifier": "BH"
                        },
                        {
                            "code": "A2",
                            "date": "20221130",
                            "qualifier": "BH"
                        },
                        {
                            "code": "45",
                            "date": "20221201",
                            "qualifier": "BH"
                        }
                    ],
                    "value_codes": [
                        {
                            "code": "01",
                            "amount": "450.00",
                            "qualifier": "BE"
                        },
                        {
                            "code": "02",
                            "amount": "600.00",
                            "qualifier": "BE"
                        },
                        {
                            "code": "30",
                            "amount": "120.00",
                            "qualifier": "BE"
                        }
                    ],
                    "condition_codes": [],
                    "diagnosis_codes": [
                        {
                            "code": "I269",
                            "qualifier": "ABK",
                            "poa": ""
                        },
                        {
                            "code": "I4891",
                            "qualifier": "ABF",
                            "poa": ""
                        },
                        {
                            "code": "E119",
                            "qualifier": "ABF",
                            "poa": ""
                        },
                        {
                            "code": "Z9911",
                            "qualifier": "ABF",
                            "poa": ""
                        }
                    ],
                    "procedure_codes": [],
                    "service_lines": [
                        {
                            "line_number": "1",
                            "revenue_code": "0120",
                            "procedure_type": "HC",
                            "procedure_code": "99231",
                            "modifiers": [],
                            "charge_amount": "15000.00",
                            "units": "10",
                            "service_date": "20221201",
                            "service_date_end": null
                        },
                        {
                            "line_number": "2",
                            "revenue_code": "0270",
                            "procedure_type": "HC",
                            "procedure_code": "85025",
                            "modifiers": [],
                            "charge_amount": "500.00",
                            "units": "5",
                            "service_date": "20221202",
                            "service_date_end": null
                        },
                        {
                            "line_number": "3",
                            "revenue_code": "0450",
                            "procedure_type": "HC",
                            "procedure_code": "99291",
                            "modifiers": [],
                            "charge_amount": "9500.00",
                            "units": "1",
                            "service_date": "20221205",
                            "service_date_end": null
                        }
                    ]
                }
            ]
        }
    ]
}
```

## License

[MIT License](https://opensource.org/licenses/MIT)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
