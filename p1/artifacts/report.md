### bug01 - <out_cap not used in encode_record>
- Category: Heap overflow
- Trigger: Passing a out_buf whose out_cap is smaller than the size required to encode the record. Example: score_count = 1000; out_cap = 10; encode_record(&rec, out_buf, 10); will then overflow the heap buffer.
- Root casue: After calculates the space needed to put into out_buf, encode_record never checks if the out_cap is large enough for the space needed before writting into out_buf. if out_cap < space, then iw will cause a heap buffer overflow. the function should not assumes the caller always provides a sufficiently large buffer.
- Fix: Add a proper check that checks the out_cap > space after computed the space needed, if not return 0. This ensures that the function aborts instead of overflowing the buffer

### bug02 - <Heap buffer overflow in decode_record name field>
- Category: Heap overflow
- Trigger: The record_t struct defines name as char name[32]. However, name_len is a uint8_t, meaning it can be up to 255. If name_len is 100, line32 copies 100 bytes into a 32byte container, causing a Heap Buffer Overflow.
- Root casue: decode_record allocates record_t on the heap, then copies name_len bytes into the fixed-size rec->name[32] array without validating length. THis ensures the buffer actually contains the claimed name_len bytes, so there will be no overflow.

### bug03 - <Out-of-bounds read from input buffer>
- Category: Out-of-bounds read
- Trigger: Any record where name_len is smaller than MAX_NAME but greater than the remaining bytes in the input buffer. Example, input only have name_len and 2 ytes after it.
- Root casue: decode_record copies name_len bytes from buf without checking if enough bytes remain. If off + name_len > len, this causes an out-of-bounds read past the end of buf.
- Fix: Before copying, ensure the buffer contains all required bytes. This prevents reading outside the input buffer.

### bug04 - <Integer overflow in score_count leads to heap overflow>
- Category: Integer overflow, heap overflow
- Trigger: An encoded record where score_count (4-byte field after age) is very large. Example: 0xFF 0xFF 0xFF 0xFF
- Root casue: In decode_record, score_count is read directly from the input and used in a multiplication without any validation. If score_count is extremely large, rec->score_count * 2 can overflow size_t, causing malloc to allocate a much smaller buffer than intended. Then memcpy writes space bytes into thaT small allocation, resulting in a heap overflow.
- Fix: Validate score_count before using it, reject records where score_count is above a safe maximum. Also ensure off + space <= len before memcpy. This prevents integer wraparound and ensures the allocated buffer is large enough, eliminating the heap overflow.

### bug05 - <NULL pointer dereference in scores allocation>
- Category: NULL Pointer Dereference
- Trigger: A record with a valid but large score_count that causes malloc to fail due to not enough memory.
- Root casue: In decode_record, memory is allocated for rec->scores using malloc(space). Immediately after, memcpy(rec->scores, ...) is called. There is no check to see if malloc returned NULL. If the allocation fails, memcpy attempts to write to the NULL address, crashing the program.
- Fix: Check the return value of malloc. If rec->scores is NULL, free the previously allocated rec struct and return 0 so that memcpy will not write to NULL address.

### bug06 - <Buffer overread in encode_record due to unbounded strlen>
- Category: Stack/Heap overread
- Trigger: Calling encode_record with a record_t where the name array is fully filled (32 characters) and does not contain a null-terminator \0.
- Root casue: encode_record calculates size_t name_len = strlen(rec->name);. The strlen function assumes the string is null-terminated. Since rec->name is a fixed-size array inside a struct, if the null terminator is missing, strlen will continue reading past the end of the name array into adjacent memory (stack or heap) until it crashes or finds a random /0 byte.
- Fix: Use strnlen(rec->name, MAX_NAME - 1), this forces the length to never exceed 32.
