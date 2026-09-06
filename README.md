# HARNESS

A no frills agent harness applying UNIX philosophy.

Harness does not bundle an interactive frontend nor a context backend.\
It does only one thing:\
_loop turns until done then exit._

Harness is written in bash, and uses `curl` and `jq` only.

## Usage

Install:

```bash
curl -o harness https://raw.githubusercontent.com/telamon/harness/refs/tags/v1.0.0/harness
```

```
echo "hello" | harness
Hello, how can i help?
```

Helptext:

```bash
harness -h
usage: harness [OPTIONS] < INPUT

Output:

  -v                    Print tool calls & status to STDERR
  -vv                   Print 'thinking' & 'system' type messages to STDERR
  -vvv                  Print harness & turn lifecycle events to STDERR

  -r|--raw-output       Prints all messages to STDOUT as line-delimited JSON.
                        Messages from -v are rerouted to STDOUT

  -l                    Buffer -r messages until a different role is encountered

Control:

  -q                    Confirm before each tool call
  -T                    Disable all tool calls
  -m|--max-turns 50     Limit maximum turns, 0 = unlimited

  -u|--url URL          Same as MODEL_URL
  -n|--model MODEL      Same as MODEL_NAME

  -W|--tool-bin BINARY  Run BINARY on tool-call, default: `bash -c COMMAND`

  -U|--request-bin BINARY   Use BINARY instead of built-in
                            curl command

Context:

  -s|--session FILE     Load FILE at turn start; save context at turn end.
                        (Same as: -b "cat FILE" -e "cat > FILE")

  -b|--load-ctx BINARY  Replace context with BINARY stdout at turn start,
                        before injection and prompt.

  -e|--save-ctx BINARY  Send current context to BINARY STDIN at turn end.

  -i|--inject BINARY    Call BINARY and insert output as a
                        sticky "system" message before "user" prompt.
                        May be specified multiple times.

  -X|--inject-ctx BINARY    Call BINARY and inject context;
                            See readme section 6.2.
                            May be specified multiple times.

  -I                    Cause -i and -X to be called and updated
                        before each turn.

   Note: A failing BINARY terminates harness.

Input:

  All prompt input is read from stdin.

  -P|--raw-prompt       Treat stdin as JSON-array containing
                        messages in form of:
                        `{ role: "foo", content:"bar" }`

  -R|--raw-input        Treat input as a fully assembled JSON context.
                        Cannot be combined with -P, -b, -s, -i, -X or -I.

Environment variables:

    MODEL_URL="http://127.0.0.1:11434/api/chat"
    MODEL_NAME="qwen3.5:2b"

    Additionally -b and -e receive these variables,
    metrics are zero before the first model response:

    TURN=integer
    INPUT_TOKENS=integer
    INPUT_DURATION=int_nanos
    OUTPUT_TOKENS=integer
    OUTPUT_DURATION=int_nanos

Example use:

    $ harness <<\.
    > Hello!
    > .
    Hello! How can I help you today?
```

### Tip

For practical use, set up an alias:

```bash
alias infer='harness -s context.json <<\.'
```

then

```
$ rob
> hey!
> What's 2+2?
> .

Two plus two is 4!

$ infer
> I forgot, what did you say?
> .

You asked for 2+2 it's still 4!
```

(alias name `infer` and delimiter `.` used as an example.)


## Details

### 1. Format

Harness internally uses the JSON format of Mozilla/ollama.
It is compatible with other services given that a translation
script is provided to `--request-bin`

### 2. Modularity

For a more sophisticated ui or prompt assembly,
you can use the following combination of options:

```
harness -rv     # Common default of TUI frontends
harness -rvvv   # Realtime Interactive UI

harness -Rr  # Turns harness into a tool-execution proxy,
                # use as part of an automated pipeline.

# external context manager
harness \
    --inject "ctxbin system" \
    --load-ctx "ctxbin load" \
    --save-ctx "ctxbin save"
```

### 3. Processing

Harness operates in stream mode.
Tool call requests are buffered and executed sequentially at the end of turn.
All other output is yielded on arrival.

Context is held in ram for the duration of the process.

### 4. Portability

Harness runs wherever bash `>=4.1` is available (requires `coproc` & `exec {fd}`)

`curl` can be replaced using option `-U` or setting environment variable `HARNESS_REQUESTBIN`

Internally it is defaulted to the equivalent of:
```
export HARNESS_REQUESTBIN='curl \
    --fail-with-body --silent --show-error --no-buffer \
    -H Content-Type:application/json \
    --data-binary @-'
```

And then executed as

```
$HARNESS_REQUESTBIN "$MODEL_URL"
```

(It's the planned escape hatch for api translation.)

`jq` dependency cannot currently be substituted.

`cat` and `tee` are also expected to be present.

### 5. Security

No permission layers included.
The shell environment dictates what a tool can and cannot do.

**Recommendation**: run harness in a sandbox such as `bwrap` or `jail`.

A tool call wrapper can give fine-grained control
of calls:

```
$ harness --tool-bin='echo' <<\.
> Run `sudo ls -l /`
> What do you see?
> .
I ran `sudo ls -l /` but **no actual file listing was shown**. Instead, I only see:

$ sudo ls -l /
EXIT STATUS: 0
```

but it does not provide the safety of an outer sandbox.


### 6. Tools

#### 6.1. Scripts are Tools

In a perfect world, Harness does not maintain a tool registry.

The preferred way to expose a tool is to install an executable or script in
the harness process’ `PATH`.

The only built-in `exec_command` tool invokes commands through the configured tool binary (by default, `bash -c`) and uses normal shell path resolution.

Each executable is the authority for its own interface, documentation,
validation, output, and exit status. Tools should provide useful `--help`
output and fail with a meaningful non-zero status when invoked incorrectly.

however - the world also contains stateful clients see next section (6.2)

#### 6.2. Stateful Injection

`status: draft`

Option `-X|--inject-ctx BINARY` runs `BINARY` and injects its output messages & tool sections into the request.

The option may be specified more than once; injectors run in command-line order.

Option `-I` reruns every injector before each model turn. Without `-I`,
injectors run only _once_ before the first turn.
Option `-R` cannot be combined with context injections.

##### 6.2.1. Registration
`status: draft`

Injector `stdout` is expected to return one compact JSON envelope

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "blender__get_scene",
        "description": "Return the active Blender scene.",
        "parameters": {
          "type": "object",
          "properties": {}
        }
      }
    }
  ],

  "messages": [
    { "role": "system", "sticky": true, "content": "You are operating blender" },
    { "role": "system", "content": "Selected body is `Cube.0003`" }
  ]
}
```

During registration, harness runs the injector with no stdin and `HARNESS_INIT=1` set.

**Messages**

Property `"messages"`, when present, supplies context messages.
On the first turn, messages are inserted before the user prompt; on later turns,
non-sticky messages (`"sticky"` absent or `false`) are appended at the
context tail before the next model request.

Sticky messages
(`"sticky": true`) are inserted at top of context.
If `-I` is used, they're updated on each turn rather than appended.

Property `"sticky"` is not forwarded to the model request.

_Note:_ Use non-sticky and `-I` with care;
Such entries are retained in context and may cause significat token use.

**Tools**

Property `"tools"`, when present, is merged into the next request's tool definitions. It does not enter the conversation context.

Tool names must be unique across all injectors and `exec_command` or harness will terminate.

With `-I`, each refresh replaces the complete set of injector-provided
tools. A tool no longer emitted by its owner is no longer available on
the next turn.

##### 6.2.2. Tool-call delegation
`status: draft`

Every tool returned by an injector is owned by that injector.
When the model invokes such a tool, harness runs its owner with `HARNESS_CALL=1` set and passes the complete tool-call object to its standard input:

```
{"function":{"name":"blender__get_scene","arguments":{}}}
```

The tool owner's standard output becomes the tool result and its exit status becomes the tool-result status.

Injector callback table

| BINARY call | `HARNESS_INIT` | `HARNESS_CALL` | input     | output      |
|-------------|---------------:|---------------:|-----------|-------------|
| register    |              1 |              0 | none      | spec        |
| tool-call   |              0 |              1 | tool-call | tool-result |


The complexity of this section is due to interfacing with a stateful
application - harness itself is intentionally stateless.

When in doubt, prefer a trivial scripts or programs (6.1.) over flag `-X`.

## License

GNU GPL version 3

All Wrongs Reversed - Tony Ivanov - 2026
