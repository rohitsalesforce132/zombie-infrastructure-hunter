# CONTRIBUTING.md

## How to Contribute

### Adding New Query Files

1. Create `queries/your-query.md` with Azure CLI commands
2. Include: description, commands, expected output, zombie signal
3. Test commands against a test subscription

### Adding New Detection Patterns

1. Document the pattern in `engine/gap-detector.md`
2. Include: detection method, zombie signal strength, false positive patterns
3. Add queries to support the detection

### Adding New Templates

1. Create template in `templates/`
2. Include: all fields needed for the finding
3. Keep it Markdown

### Reporting Bugs

Open an issue with:
- Azure CLI command that failed
- Expected vs actual output
- Resource type involved

## Quality Standards

- All queries must be Azure CLI commands (not PowerShell)
- All output formats must include `-o table` or `-o json`
- Cost estimates must include the calculation basis
- False positives must be documented

## License

All contributions are licensed under MIT.
