BEFORE (Your Repository):
─────────────────────────
repo/
├── main.sas                 ← Contains: %INCLUDE 'utils/setup.sas';
├── utils/
│   ├── setup.sas            ← Contains: %INCLUDE 'macros.sas';
│   └── macros.sas
└── cleanup.sas

AFTER (Split Output):
─────────────────────
output/
├── main_split.sas           ← %INCLUDE stays as a chunk (reference)
├── main_metadata.json       ← Links to dependent files
├── utils/
│   ├── setup_split.sas      ← %INCLUDE stays as a chunk
│   ├── setup_metadata.json
│   ├── macros_split.sas
│   └── macros_metadata.json
├── cleanup_split.sas
├── cleanup_metadata.json
└── dependency_graph.json    ← Shows include relationships






PROMPT FOR CLAUDE CODE:
-----------------------
Modify the SAS splitter to handle %INCLUDE statements WITHOUT inlining 
the included content. Each file gets its own split output.

Requirements:

1. When encountering %INCLUDE, treat it as a CHUNK of type INCLUDE:
   
   @dataclass
   class IncludeChunk:
       chunk_id: int
       chunk_type: ChunkType  # = ChunkType.INCLUDE
       code: str              # The original %INCLUDE statement
       start_line: int
       end_line: int
       
       # Include-specific fields
       include_path: str              # Path as written in code
       resolved_path: Optional[str]   # Full resolved path (if found)
       is_resolved: bool              # Whether file was found
       target_split_file: Optional[str]  # Path to the split file for this include

2. The splitter should NOT:
   - Read the included file's content into the current file
   - Inline chunks from included files
   - Modify the %INCLUDE statement

3. The splitter SHOULD:
   - Detect %INCLUDE statements
   - Resolve the file path (to verify it exists)
   - Create a chunk for the %INCLUDE itself
   - Record the dependency relationship
   - Continue splitting the current file

4. Example output for main.sas containing:
   
   data step1; set x; run;
   %INCLUDE 'utils/setup.sas';
   proc sql; create table y as select * from z; quit;
   
   Should produce 3 chunks:
   
   Chunk 1: DATA_STEP (data step1...)
   Chunk 2: INCLUDE (%INCLUDE 'utils/setup.sas';)
            - include_path: 'utils/setup.sas'
            - resolved_path: '/full/path/utils/setup.sas'
            - target_split_file: 'utils/setup_split.sas'
   Chunk 3: PROC_SQL (proc sql...)

5. Regex pattern for %INCLUDE (capture but don't follow):
   
   INCLUDE_PATTERN = regex.compile(r'''
       (?P<full_statement>
           %INCLUDE\s+
           (?:
               '(?P<single_quoted>[^']+)'   |
               "(?P<double_quoted>[^"]+)"   |
               (?P<libref>\w+)\s*\((?P<member>\w+)\)  |
               (?P<unquoted>[^\s;]+)
           )
           \s*
           (?:/[^;]*)?     # Optional options like /SOURCE2
           \s*;
       )
   ''', regex.VERBOSE | regex.IGNORECASE)



















PROMPT FOR CLAUDE CODE:
-----------------------
Create a BatchSplitter class that processes multiple SAS files and 
resolves %INCLUDE dependencies between them.

Requirements:

1. BatchSplitter class:
   
   class BatchSplitter:
       def __init__(
           self,
           input_dir: str,           # Root directory of SAS files
           output_dir: str,          # Where to write split files
           base_paths: List[str] = None,  # Additional search paths
           recursive: bool = True    # Search subdirectories
       ):
           self.input_dir = input_dir
           self.output_dir = output_dir
           self.base_paths = base_paths or []
           self.recursive = recursive
           self.splitter = SASCodeSplitter()
           
           # Dependency tracking
           self.file_dependencies: Dict[str, Set[str]] = {}  # file -> set of includes
           self.processed_files: Dict[str, SplitResult] = {}
           self.unresolved_includes: Dict[str, List[str]] = {}

2. Main processing method:
   
   def process_all(self, entry_point: str = None) -> BatchResult:
       """
       Process all SAS files starting from entry_point.
       If entry_point is None, discover and process all .sas files.
       
       Processing order:
       1. If entry_point given, start there and follow includes
       2. Otherwise, find all .sas files and process in dependency order
       """
       
       if entry_point:
           # Process starting from entry point, following includes
           return self._process_from_entry_point(entry_point)
       else:
           # Discover all files and process
           return self._process_all_discovered()
   
   def _process_from_entry_point(self, entry_file: str) -> BatchResult:
       """
       Process entry file, then recursively process any %INCLUDE files.
       """
       to_process = [entry_file]
       processing_order = []
       
       while to_process:
           current_file = to_process.pop(0)
           
           if current_file in self.processed_files:
               continue
           
           # Split this file
           result = self._process_single_file(current_file)
           self.processed_files[current_file] = result
           processing_order.append(current_file)
           
           # Find includes and add to queue
           for chunk in result.chunks:
               if chunk.chunk_type == ChunkType.INCLUDE:
                   if chunk.resolved_path and chunk.is_resolved:
                       if chunk.resolved_path not in self.processed_files:
                           to_process.append(chunk.resolved_path)
       
       return BatchResult(
           processed_files=self.processed_files,
           processing_order=processing_order,
           dependency_graph=self.file_dependencies,
           unresolved=self.unresolved_includes
       )

3. Single file processing:
   
   def _process_single_file(self, file_path: str) -> SplitResult:
       """
       Split a single file and resolve its include references.
       """
       # Read and split
       with open(file_path, 'r') as f:
           code = f.read()
       
       result = self.splitter.split(code, source_file=file_path)
       
       # Process INCLUDE chunks - resolve paths
       dependencies = set()
       for chunk in result.chunks:
           if chunk.chunk_type == ChunkType.INCLUDE:
               resolved = self._resolve_include_path(
                   chunk.include_path,
                   base_dir=os.path.dirname(file_path)
               )
               chunk.resolved_path = resolved
               chunk.is_resolved = resolved is not None
               
               if resolved:
                   # Calculate relative path for target split file
                   chunk.target_split_file = self._get_split_file_path(resolved)
                   dependencies.add(resolved)
               else:
                   # Track unresolved
                   if file_path not in self.unresolved_includes:
                       self.unresolved_includes[file_path] = []
                   self.unresolved_includes[file_path].append(chunk.include_path)
       
       self.file_dependencies[file_path] = dependencies
       
       # Write split file
       self._write_split_output(file_path, result)
       
       return result

4. Output file path calculation:
   
   def _get_split_file_path(self, source_path: str) -> str:
       """
       Calculate output path maintaining directory structure.
       
       Input:  /repo/utils/setup.sas
       Output: /output/utils/setup_split.sas
       """
       # Get relative path from input_dir
       rel_path = os.path.relpath(source_path, self.input_dir)
       
       # Change extension
       base, _ = os.path.splitext(rel_path)
       split_name = f"{base}_split.sas"
       
       return os.path.join(self.output_dir, split_name)




























PROMPT FOR CLAUDE CODE:
-----------------------
Define the split file output format that preserves %INCLUDE as chunks.

Requirements:

1. Split file format for a file containing %INCLUDE:
   
   /* ================================================================ */
   /* SPLIT FILE: main.sas                                             */
   /* Generated: 2024-01-15T10:30:00                                   */
   /* Original: /repo/main.sas                                         */
   /* ================================================================ */
   /* DEPENDENCIES:                                                    */
   /*   - utils/setup.sas -> utils/setup_split.sas                    */
   /*   - cleanup.sas -> cleanup_split.sas                            */
   /* ================================================================ */
   
   /* --- PART 000001 START --- */
   /* Type: DATA_STEP | Outputs: WORK.STEP1 | Inputs: WORK.X */
   data step1; 
       set x; 
   run;
   /* --- PART 000001 END --- */
   
   /* --- PART 000002 START --- */
   /* Type: INCLUDE | Target: utils/setup_split.sas */
   /* NOTE: Process utils/setup_split.sas before continuing */
   %INCLUDE 'utils/setup.sas';
   /* --- PART 000002 END --- */
   
   /* --- PART 000003 START --- */
   /* Type: PROC_SQL | Outputs: WORK.Y | Inputs: WORK.Z */
   proc sql; 
       create table y as 
       select * from z; 
   quit;
   /* --- PART 000003 END --- */

2. Metadata JSON for file with includes:
   
   {
     "source_file": "/repo/main.sas",
     "split_file": "/output/main_split.sas",
     "generated": "2024-01-15T10:30:00",
     
     "dependencies": {
       "includes": [
         {
           "chunk_id": 2,
           "original_path": "utils/setup.sas",
           "resolved_path": "/repo/utils/setup.sas",
           "split_file": "utils/setup_split.sas",
           "is_resolved": true
         }
       ],
       "required_before_execution": [
         "utils/setup_split.sas"
       ]
     },
     
     "chunks": [
       {
         "chunk_id": 1,
         "chunk_type": "DATA_STEP",
         "start_line": 1,
         "end_line": 3,
         "output_tables": ["WORK.STEP1"],
         "input_tables": ["WORK.X"]
       },
       {
         "chunk_id": 2,
         "chunk_type": "INCLUDE",
         "start_line": 4,
         "end_line": 4,
         "include_path": "utils/setup.sas",
         "resolved_path": "/repo/utils/setup.sas",
         "target_split_file": "utils/setup_split.sas",
         "is_resolved": true,
         "output_tables": [],
         "input_tables": []
       },
       {
         "chunk_id": 3,
         "chunk_type": "PROC_SQL",
         "start_line": 5,
         "end_line": 8,
         "output_tables": ["WORK.Y"],
         "input_tables": ["WORK.Z"]
       }
     ],
     
     "summary": {
       "total_chunks": 3,
       "include_chunks": 1,
       "data_step_chunks": 1,
       "proc_chunks": 1
     }
   }

3. Generate method:
   
   def generate_split_file_with_includes(self, result: SplitResult) -> str:
       """Generate split file content preserving %INCLUDE references."""
       
       lines = []
       
       # Header
       lines.append("/* " + "=" * 64 + " */")
       lines.append(f"/* SPLIT FILE: {os.path.basename(result.original_file)}")
       lines.append(f"/* Generated: {datetime.now().isoformat()}")
       lines.append(f"/* Original: {result.original_file}")
       lines.append("/* " + "=" * 64 + " */")
       
       # Dependencies section
       include_chunks = [c for c in result.chunks if c.chunk_type == ChunkType.INCLUDE]
       if include_chunks:
           lines.append("/* DEPENDENCIES:")
           for ic in include_chunks:
               if ic.is_resolved:
                   lines.append(f"/*   - {ic.include_path} -> {ic.target_split_file}")
               else:
                   lines.append(f"/*   - {ic.include_path} -> UNRESOLVED")
           lines.append("/* " + "=" * 64 + " */")
       
       lines.append("")
       
       # Chunks
       for chunk in result.chunks:
           part_num = f"{chunk.chunk_id:06d}"
           
           lines.append(f"/* --- PART {part_num} START --- */")
           
           if chunk.chunk_type == ChunkType.INCLUDE:
               # Special handling for INCLUDE chunks
               lines.append(f"/* Type: INCLUDE | Target: {chunk.target_split_file or 'UNRESOLVED'} */")
               if chunk.is_resolved:
                   lines.append(f"/* NOTE: Process {chunk.target_split_file} at this point */")
               else:
                   lines.append(f"/* WARNING: Include file not found: {chunk.include_path} */")
           else:
               # Regular chunk
               outputs = ', '.join(chunk.output_tables) or 'None'
               inputs = ', '.join(chunk.input_tables) or 'None'
               lines.append(f"/* Type: {chunk.chunk_type.name} | Outputs: {outputs} | Inputs: {inputs} */")
           
           lines.append(chunk.code)
           lines.append(f"/* --- PART {part_num} END --- */")
           lines.append("")
       
       return '\n'.join(lines)

























PROMPT FOR CLAUDE CODE:
-----------------------
Create a module to generate a complete dependency graph for all processed files.

Requirements:

1. Create dependency_graph.json at the root of output:
   
   {
     "generated": "2024-01-15T10:30:00",
     "root_files": ["main.sas"],  # Entry points (files not included by others)
     
     "files": {
       "main.sas": {
         "split_file": "main_split.sas",
         "metadata_file": "main_metadata.json",
         "includes": ["utils/setup.sas", "cleanup.sas"],
         "included_by": [],
         "depth": 0
       },
       "utils/setup.sas": {
         "split_file": "utils/setup_split.sas",
         "metadata_file": "utils/setup_metadata.json",
         "includes": ["utils/macros.sas"],
         "included_by": ["main.sas"],
         "depth": 1
       },
       "utils/macros.sas": {
         "split_file": "utils/macros_split.sas",
         "metadata_file": "utils/macros_metadata.json",
         "includes": [],
         "included_by": ["utils/setup.sas"],
         "depth": 2
       },
       "cleanup.sas": {
         "split_file": "cleanup_split.sas",
         "metadata_file": "cleanup_metadata.json",
         "includes": [],
         "included_by": ["main.sas"],
         "depth": 1
       }
     },
     
     "execution_order": [
       "utils/macros.sas",    # Depth 2 - no dependencies
       "utils/setup.sas",     # Depth 1 - depends on macros
       "cleanup.sas",         # Depth 1 - no dependencies
       "main.sas"             # Depth 0 - depends on setup, cleanup
     ],
     
     "unresolved_includes": {
       "main.sas": ["missing_file.sas"]
     },
     
     "statistics": {
       "total_files": 4,
       "total_chunks": 25,
       "max_depth": 2,
       "unresolved_count": 1
     }
   }

2. DependencyGraphBuilder class:
   
   class DependencyGraphBuilder:
       def __init__(self, batch_result: BatchResult):
           self.batch_result = batch_result
           self.graph = {}
           self.reverse_graph = {}  # included_by relationships
       
       def build(self) -> Dict:
           """Build the complete dependency graph."""
           
           # Build forward graph (includes)
           for file_path, result in self.batch_result.processed_files.items():
               includes = self._get_includes(result)
               self.graph[file_path] = includes
               
               # Build reverse graph
               for included in includes:
                   if included not in self.reverse_graph:
                       self.reverse_graph[included] = []
                   self.reverse_graph[included].append(file_path)
           
           # Calculate depths
           depths = self._calculate_depths()
           
           # Determine execution order (topological sort)
           execution_order = self._topological_sort()
           
           # Find root files (not included by anyone)
           root_files = [f for f in self.graph if f not in self.reverse_graph]
           
           return {
               "generated": datetime.now().isoformat(),
               "root_files": root_files,
               "files": self._build_files_dict(depths),
               "execution_order": execution_order,
               "unresolved_includes": self.batch_result.unresolved_includes,
               "statistics": self._calculate_statistics(depths)
           }
       
       def _topological_sort(self) -> List[str]:
           """
           Return files in order such that dependencies come before dependents.
           Leaf files (no includes) come first, root files (not included) come last.
           """
           # Kahn's algorithm
           in_degree = {f: 0 for f in self.graph}
           for includes in self.graph.values():
               for inc in includes:
                   if inc in in_degree:
                       in_degree[inc] += 1
           
           # Start with files that aren't included by anyone
           queue = [f for f, degree in in_degree.items() if degree == 0]
           result = []
           
           while queue:
               current = queue.pop(0)
               result.append(current)
               
               for included in self.graph.get(current, []):
                   if included in in_degree:
                       in_degree[included] -= 1
                       if in_degree[included] == 0:
                           queue.append(included)
           
           # Reverse so dependencies come first
           return result[::-1]
















PROMPT FOR CLAUDE CODE:
-----------------------
Add CLI commands for batch processing with include handling.

Requirements:

1. New CLI commands:
   
   # Process single file (don't follow includes, just mark them)
   python main.py split input.sas --output-dir ./output
   
   # Process file AND all its includes (each to separate split file)
   python main.py split-with-deps input.sas --output-dir ./output
   
   # Process entire directory
   python main.py split-all ./repo --output-dir ./output --recursive
   
   # Generate dependency graph only (no splitting)
   python main.py deps input.sas --output deps.json

2. Add to argparser:
   
   # split-with-deps command
   deps_parser = subparsers.add_parser(
       'split-with-deps', 
       help='Split file and all its %INCLUDE dependencies'
   )
   deps_parser.add_argument('input_file', help='Entry point SAS file')
   deps_parser.add_argument('--output-dir', '-o', default='./output')
   deps_parser.add_argument(
       '--base-paths', 
       nargs='+', 
       help='Additional directories to search for includes'
   )
   deps_parser.add_argument(
       '--max-depth', 
       type=int, 
       default=10,
       help='Maximum include nesting depth'
   )
   
   # split-all command
   all_parser = subparsers.add_parser(
       'split-all',
       help='Split all SAS files in directory'
   )
   all_parser.add_argument('input_dir', help='Directory containing SAS files')
   all_parser.add_argument('--output-dir', '-o', default='./output')
   all_parser.add_argument(
       '--recursive', '-r',
       action='store_true',
       help='Search subdirectories'
   )
   all_parser.add_argument(
       '--entry-point',
       help='Optional: Start from this file and follow includes'
   )

3. Command handlers:
   
   def cmd_split_with_deps(self, args) -> int:
       """Process file and all dependencies."""
       
       print(f"\n{'='*60}")
       print("SAS SPLITTER - Processing with Dependencies")
       print(f"{'='*60}")
       print(f"Entry point: {args.input_file}")
       print(f"Output: {args.output_dir}")
       
       batch = BatchSplitter(
           input_dir=os.path.dirname(args.input_file),
           output_dir=args.output_dir,
           base_paths=args.base_paths or []
       )
       
       result = batch.process_all(entry_point=args.input_file)
       
       # Print summary
       print(f"\n--- Processing Summary ---")
       print(f"Files processed: {len(result.processed_files)}")
       print(f"Processing order:")
       for i, f in enumerate(result.processing_order, 1):
           print(f"  {i}. {f}")
       
       if result.unresolved:
           print(f"\n⚠️  Unresolved includes:")
           for file, includes in result.unresolved.items():
               for inc in includes:
                   print(f"  - {file}: {inc}")
       
       # Write dependency graph
       graph_path = os.path.join(args.output_dir, 'dependency_graph.json')
       with open(graph_path, 'w') as f:
           json.dump(result.dependency_graph, f, indent=2)
       print(f"\nDependency graph: {graph_path}")
       
       print(f"{'='*60}\n")
       return 0








YOUR APPROACH (Separate Files):
───────────────────────────────

Input:                          Output:
──────                          ───────

main.sas ─────────────────────► main_split.sas
│                               │ Chunk 1: DATA step
│ (line 10)                     │ Chunk 2: %INCLUDE 'setup.sas'  ──┐
│ %INCLUDE 'setup.sas'; ────┐   │ Chunk 3: PROC SQL               │
│                           │   │                                  │
│ (line 20)                 │   main_metadata.json                 │
│ %INCLUDE 'cleanup.sas'; ──┼─┐   └─► "dependencies": [setup, cleanup]
│                           │ │                                    │
└───────────────────────────│─│───────────────────────────────────┘
                            │ │
setup.sas ◄─────────────────┘ │─► setup_split.sas
│                             │   │ Chunk 1: %LET group
│ %INCLUDE 'macros.sas'; ─────│─┐ │ Chunk 2: %INCLUDE 'macros.sas'
│                             │ │ │ Chunk 3: DATA step
└─────────────────────────────│─│─│
                              │ │ │
cleanup.sas ◄─────────────────┘ │ └► cleanup_split.sas
│                               │     │ Chunk 1: PROC DATASETS
│ (no includes)                 │     │ Chunk 2: DATA step
└───────────────────────────────│
                                │
macros.sas ◄────────────────────┘──► macros_split.sas
│                                    │ Chunk 1: %MACRO def
│ (no includes)                      │ Chunk 2: %MACRO def
└──────────────────────────────────


dependency_graph.json:
─────────────────────
{
  "execution_order": [
    "macros.sas",      ← Process first (no deps)
    "setup.sas",       ← Depends on macros
    "cleanup.sas",     ← No deps
    "main.sas"         ← Depends on setup, cleanup
  ]
}


















   
   
