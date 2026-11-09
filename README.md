# portfolio-ai

Built with the [Ralph](https://github.com/snarktank/ralph) autonomous agent loop: an AI coding tool runs repeatedly, one user story per iteration, until every story in `prd.json` passes.

## Quick start

1. Write a PRD and convert it to `prd.json` (see `prd.json.example` and the skills in `skills/`).
2. Run the loop with Claude Code:

   ```bash
   ./ralph.sh --tool claude 10
   ```

Progress is logged to `progress.txt`; both it and `prd.json` are git-ignored working files.

See [RALPH.md](RALPH.md) for the full upstream documentation. Ralph is MIT-licensed by snarktank (see `LICENSE`).
