1. Python's regex Module (Matthew Barnett's)

Supports recursive patterns via (?R) or (?1), (?2) for specific groups
Install: pip install regex
Can handle nested structures like %MACRO containing %MACRO
Example: r'%MACRO\s+\w+.*?(?:%MACRO.*?%MEND|(?R))*.*?%MEND'

2. State Machine + Regex Hybrid Approach

Best practice for complex parsing: use regex for tokenization, state machine for structure
Track context: Are we inside a string? Comment? Macro? Datalines?
This prevents false matches inside strings/comments

3. Nested Macro Variable Resolution

&var → single resolution
&&var → & + value of var (two-pass)
&&&var → & + & + value of var (three-pass)
Pattern: r'(&+)([A-Za-z_]\w*)(\.)?' captures nesting level by counting &



Phase 1: Core Infrastructure


PROMPT FOR CLAUDE CODE:
-----------------------
Create a Python module `sas_tokenizer.py` that tokenizes SAS code into 
meaningful tokens. Use the `regex` module (pip install regex) for 
recursive pattern support.

Requirements:
1. Define token types as an Enum: MACRO_START, MACRO_END, DATA_START, 
   PROC_START, RUN, QUIT, LET, INCLUDE, COMMENT, STRING, SEMICOLON, 
   MACRO_VAR, IDENTIFIER, OTHER

2. Create a Tokenizer class that:
   - Tracks parser state (normal, in_string, in_comment, in_datalines)
   - Returns tokens with: type, value, line_number, column
   - Handles SAS's case-insensitivity
   - Preserves string literals and comments without parsing inside them

3. Key regex patterns to implement:
   - Block comments: r'/\*[\s\S]*?\*/'
   - Line comments: r'\*[^;]*;'
   - Single-quoted strings: r"'(?:''|[^'])*'"
   - Double-quoted strings: r'"(?:""|[^"])*"'
   - Macro variables: r'(&+)([A-Za-z_]\w*)(\.)?'
   - %LET: r'%LET\s+(\w+)\s*=\s*([^;]*);'
   - %INCLUDE: r'%INCLUDE\s+["\']?([^"\';\s]+)["\']?\s*;'
   - %MACRO: r'%MACRO\s+(\w+)(?:\s*\(([^)]*)\))?\s*;'
   - %MEND: r'%MEND\s*(\w*)\s*;'
   - DATA step: r'DATA\s+([^;]+);'
   - PROC: r'PROC\s+(\w+)'

4. Handle datalines/cards sections specially (don't parse until lone semicolon)











Phase 2: %INCLUDE Handler
PROMPT FOR CLAUDE CODE:
-----------------------
Create a module `include_resolver.py` that handles %INCLUDE statements.

Requirements:
1. Create an IncludeResolver class that:
   - Takes a base directory path for resolving relative includes
   - Maintains a set of already-included files (prevent infinite loops)
   - Tracks include depth (max depth = 10 recommended)
   - Supports SAS path formats:
     * %INCLUDE 'path/to/file.sas';
     * %INCLUDE "path/to/file.sas";
     * %INCLUDE fileref(member);  -- library reference style

2. resolve_includes(code: str, base_path: str) -> str method:
   - Find all %INCLUDE statements
   - For each, read the included file
   - Recursively resolve includes in that file
   - Replace %INCLUDE with actual content (wrapped in markers)
   - Add markers: /* --- INCLUDED FROM: filename START --- */ and END

3. Handle errors gracefully:
   - File not found: log warning, leave %INCLUDE in place with comment
   - Circular includes: detect and break the cycle with warning
   - Return metadata about all included files

4. Regex pattern for includes:
   %INCLUDE\s+(?:
     '([^']+)'|          # Single quoted path
     "([^"]+)"|          # Double quoted path
     (\w+)\s*\((\w+)\)|  # Library reference: libref(member)
     ([^\s;]+)           # Unquoted path
   )\s*;


   Phase 3: %LET Statement Grouper
PROMPT FOR CLAUDE CODE:
-----------------------
Create logic in the splitter to group consecutive %LET statements.

Requirements:
1. When parsing, detect sequences of consecutive %LET statements
   (allowing only whitespace/comments between them)

2. Group them into a single LET_GROUP chunk type

3. Extract macro variable definitions from each %LET:
   - Variable name
   - Value (may contain macro variables itself)
   - Track if value references other macro variables

4. Regex pattern:
   r'%LET\s+(\w+)\s*=\s*((?:[^;]|;(?=\s*%LET))*);'
   
   This captures LET statements and allows semicolons if followed by another %LET

5. Store grouped %LETs with metadata:
   {
     "chunk_type": "LET_GROUP",
     "variables": [
       {"name": "var1", "value": "value1", "references": ["other_var"]},
       {"name": "var2", "value": "&var1._suffix", "references": ["var1"]}
     ]
   }

Phase 4: Hierarchical %MACRO Parser (2nd Order Chunks)

PROMPT FOR CLAUDE CODE:
-----------------------
Create a module `macro_parser.py` for hierarchical macro parsing.

Requirements:
1. Use the `regex` module's recursive pattern to match nested macros:
   
   MACRO_PATTERN = regex.compile(r'''
     %MACRO \s+ (\w+)           # Macro name
     (?:\s*\(([^)]*)\))?        # Optional parameters  
     \s* ;                       # Opening semicolon
     (                           # Macro body - GROUP 3
       (?:
         [^%]+ |                 # Non-% characters
         %(?!MACRO|MEND) |       # % not followed by MACRO/MEND
         (?P<nested>             # Nested macro (named group for recursion)
           %MACRO \s+ \w+ (?:\s*\([^)]*\))? \s* ;
           (?P>nested)?          # Recurse into nested
           %MEND \s* \w* \s* ;
         )
       )*
     )
     %MEND \s* \w* \s* ;        # Closing %MEND
   ''', regex.VERBOSE | regex.IGNORECASE)

2. After extracting macro body, create SUB-CHUNKS by parsing the body 
   for DATA steps and PROC steps:
   
   - Parse macro body with the regular splitter logic
   - Each DATA/PROC becomes a SubChunk with:
     * sub_chunk_id (1, 2, 3... within the macro)
     * parent_chunk_id
     * chunk_type
     * code
     * relative line numbers (within macro)

3. Handle macro-specific constructs inside body:
   - %IF/%THEN/%ELSE blocks (keep as single unit, don't split)
   - %DO loops (keep as single unit)
   - %LOCAL/%GLOBAL declarations

4. Output structure:
   {
     "chunk_id": 5,
     "chunk_type": "MACRO_DEF",
     "macro_name": "process_data",
     "parameters": ["input_ds", "output_ds"],
     "sub_chunks": [
       {
         "sub_chunk_id": 1,
         "chunk_type": "DATA_STEP",
         "code": "data &output_ds; set &input_ds; run;"
       },
       {
         "sub_chunk_id": 2, 
         "chunk_type": "PROC_SQL",
         "code": "proc sql; create table..."
       }
     ]
   }



   Phase 5: Nested Macro Variable Analyzer
PROMPT FOR CLAUDE CODE:
-----------------------
Create a module `macro_variable_analyzer.py` for analyzing macro variable usage.

Requirements:
1. Detect all macro variable references with nesting levels:
   
   MACRO_VAR_PATTERN = r'(&+)([A-Za-z_]\w*)(\.)?'
   
   - Group 1: The ampersands (count them for nesting level)
   - Group 2: Variable name
   - Group 3: Optional trailing dot (prevents concatenation issues)

2. Classify macro variable references:
   - Level 1 (&var): Direct reference
   - Level 2 (&&var): Delayed reference (resolves to &value)
   - Level 3+ (&&&var, &&&&var): Multiple-pass resolution

3. Build dependency graph:
   - Track which macro variables depend on others
   - Detect potential resolution order issues
   - Flag variables that can't be statically resolved

4. For each chunk, output:
   {
     "macro_variables": [
       {
         "reference": "&input_table",
         "nesting_level": 1,
         "can_resolve_statically": false,
         "depends_on": []
       },
       {
         "reference": "&&prefix&i",
         "nesting_level": 2,
         "can_resolve_statically": false,
         "depends_on": ["prefix", "i"]
       }
     ]
   }

5. Handle macro quoting functions:
   - %STR(), %NRSTR(): Contents should not be parsed for macro vars
   - %BQUOTE(), %SUPERQ(): Mark as runtime-resolved
  

Phase 6: Main Splitter Integration

PROMPT FOR CLAUDE CODE:
-----------------------
Create the main `sas_advanced_splitter.py` that integrates all modules.

Requirements:
1. SASAdvancedSplitter class with methods:
   - split(code: str, source_file: str) -> SplitResult
   - split_file(file_path: str) -> SplitResult
   - split_with_includes(file_path: str, base_dir: str) -> SplitResult

2. Processing pipeline:
   a. Resolve %INCLUDEs (recursive)
   b. Tokenize the combined code
   c. First pass: Identify major blocks (%MACRO, DATA, PROC)
   d. Group consecutive %LET statements
   e. Second pass: Parse sub-chunks within %MACROs
   f. Analyze macro variables in each chunk
   g. Track output/input tables
   h. Detect table overwrites

3. State machine for accurate splitting:
   
   States: NORMAL, IN_MACRO, IN_DATA, IN_PROC, IN_PROC_SQL, IN_COMMENT, 
           IN_STRING, IN_DATALINES
   
   Transitions:
   - NORMAL + "data " → IN_DATA
   - NORMAL + "proc sql" → IN_PROC_SQL  
   - NORMAL + "proc " → IN_PROC
   - NORMAL + "%macro " → IN_MACRO
   - IN_DATA + "run;" → NORMAL (emit chunk)
   - IN_PROC + "run;" → NORMAL (emit chunk)
   - IN_PROC_SQL + "quit;" → NORMAL (emit chunk)
   - IN_MACRO + "%mend" → NORMAL (emit chunk, then parse sub-chunks)

4. Output structure matching your existing format but with extensions:
   - Add "sub_chunks" array for MACRO_DEF types
   - Add "macro_variables" analysis
   - Add "included_from" for chunks from %INCLUDE files
   - Add "let_group_variables" for LET_GROUP types
  


Phase 7: Validation & Testing


PROMPT FOR CLAUDE CODE:
-----------------------
Create comprehensive tests in `test_advanced_splitter.py`.

Test cases needed:
1. Simple %INCLUDE (single level)
2. Nested %INCLUDE (file A includes B includes C)
3. Circular %INCLUDE detection
4. %INCLUDE with different path formats

5. Single %LET statement
6. Multiple consecutive %LET statements (should group)
7. %LET statements separated by other code (should NOT group)

8. Simple %MACRO with one DATA step
9. %MACRO with multiple DATA/PROC steps (verify sub-chunks)
10. Nested %MACRO definitions
11. %MACRO with %IF/%THEN/%ELSE blocks

12. Single & macro variable
13. Double && macro variable  
14. Triple &&& macro variable
15. Macro variables inside strings (should detect but flag)
16. Macro variables with trailing dot

17. Complex real-world SAS script combining all features

Also create a validation script that:
- Compares original code execution vs split code execution
- Verifies no code is lost or duplicated
- Checks all chunks can be reassembled to original




Consider using the Agent SDK for these specific scenarios:
Scenario 1: Intelligent %INCLUDE Path Resolution

When the splitter encounters %INCLUDE with:
- Relative paths that need context
- Library references (libref(member)) that need mapping
- Missing files that might exist elsewhere

Agent can:
- Search the codebase for likely matches
- Ask for clarification on library mappings
- Suggest corrections for typos in filenames


Scenario 2: Complex Macro Analysis

When encountering:
- Highly dynamic macros with CALL EXECUTE
- Macros that generate other macros
- %SYSFUNC with complex expressions

Agent can:
- Analyze the macro's intent
- Suggest how to handle dynamic code generation
- Flag sections needing manual review


Scenario 3: Validation Assistance
When validation shows differences:
- Agent can analyze the diff
- Suggest which chunk might have the issue
- Propose fixes for common splitting errors


Agent SDK Integration Point

# Pseudocode for agent integration
class SASSmartSplitter:
    def __init__(self, use_agent=False):
        self.use_agent = use_agent
        if use_agent:
            from anthropic_agent_sdk import Agent
            self.agent = Agent(model="claude-sonnet-4-20250514")
    
    def handle_complex_include(self, include_stmt, context):
        if self.use_agent:
            response = self.agent.query(
                f"Resolve this SAS %INCLUDE: {include_stmt}\n"
                f"Available files: {context.available_files}\n"
                f"Library mappings: {context.librefs}"
            )
            return response.resolved_path
        else:
            # Fall back to basic resolution
            return self.basic_resolve(include_stmt)







Key Regex Patterns Reference

PATTERNS = {
    # Blocks
    'macro_def': r'%MACRO\s+(\w+)(?:\s*\(([^)]*)\))?\s*;',
    'macro_end': r'%MEND\s*(\w*)\s*;',
    'data_step': r'DATA\s+([^;/]+)\s*;',
    'proc_start': r'PROC\s+(\w+)',
    'run': r'\bRUN\s*;',
    'quit': r'\bQUIT\s*;',
    
    # Macro statements
    'let': r'%LET\s+(\w+)\s*=\s*([^;]*);',
    'include': r'%INCLUDE\s+["\']?([^"\';\s]+)["\']?\s*;',
    'macro_call': r'%(\w+)\s*(?:\(([^)]*)\))?',
    
    # Macro variables (with nesting detection)
    'macro_var': r'(&+)([A-Za-z_]\w*)(\.)?',
    
    # Strings and comments (to skip)
    'single_string': r"'(?:''|[^'])*'",
    'double_string': r'"(?:""|[^"])*"',
    'block_comment': r'/\*[\s\S]*?\*/',
    'line_comment': r'\*[^;]*;',
    
    # Table references
    'table_ref': r'([A-Za-z_]\w*(?:\.[A-Za-z_]\w*)?)',
    'set_stmt': r'\bSET\s+([^;]+);',
    'merge_stmt': r'\bMERGE\s+([^;]+);',
    'from_clause': r'\bFROM\s+([A-Za-z_]\w*(?:\.[A-Za-z_]\w*)?)',
    'create_table': r'CREATE\s+TABLE\s+([A-Za-z_]\w*(?:\.[A-Za-z_]\w*)?)',
}
