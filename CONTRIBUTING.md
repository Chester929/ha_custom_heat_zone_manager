# Contributing to Floor Heating Valve Manager

Thank you for your interest in contributing to the Floor Heating Valve Manager blueprint! We welcome contributions from the community.

## How to Contribute

### Reporting Bugs

If you find a bug, please open an issue on GitHub with:

1. **Clear title** describing the problem
2. **Home Assistant version** you're using
3. **Blueprint version** (check CHANGELOG.md)
4. **Your configuration** (sanitize any sensitive data)
5. **Steps to reproduce** the issue
6. **Expected behavior** vs actual behavior
7. **Relevant logs** from Home Assistant
8. **Automation trace** if applicable

### Suggesting Enhancements

We welcome feature requests! Please open an issue with:

1. **Clear description** of the enhancement
2. **Use case** - why is this needed?
3. **Proposed solution** (if you have one)
4. **Alternative solutions** you've considered
5. **Impact** on existing functionality

### Pull Requests

We love pull requests! Here's how to submit one:

1. **Fork the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/ha_custom_heat_zone_manager.git
   cd ha_custom_heat_zone_manager
   ```

2. **Create a branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```
   Use prefixes: `feature/`, `bugfix/`, `docs/`, `refactor/`

3. **Make your changes**
   - Keep changes focused and minimal
   - Follow the existing code style
   - Update documentation if needed

4. **Test your changes**
   - Validate YAML syntax
   - Test with Home Assistant if possible
   - Ensure no breaking changes to existing configurations

5. **Commit your changes**
   ```bash
   git add .
   git commit -m "Add feature: description of your changes"
   ```
   
   Use clear, descriptive commit messages:
   - `Add: ` for new features
   - `Fix: ` for bug fixes
   - `Docs: ` for documentation
   - `Refactor: ` for code improvements

6. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```

7. **Open a Pull Request**
   - Go to the original repository
   - Click "New Pull Request"
   - Select your branch
   - Fill in the PR template

### Code Style Guidelines

#### YAML Style
- Use 2 spaces for indentation (no tabs)
- Keep lines under 120 characters when possible
- Add comments for complex logic
- Use meaningful variable names
- Group related configuration together

#### Documentation Style
- Use clear, simple English
- Include examples for complex features
- Keep documentation up-to-date with code changes
- Use markdown formatting properly
- Add screenshots for UI-related changes

#### Template Style
- Use descriptive template variable names
- Comment complex Jinja2 templates
- Prefer readability over brevity
- Handle edge cases (null, undefined, empty)
- Use appropriate default values

### Testing

Before submitting a PR:

1. **Validate YAML**
   ```bash
   python3 -c "import yaml, re; content = open('heat_zone_manager.yaml').read(); yaml.safe_load(re.sub(r'!input\s+(\w+)', r'\"\1\"', content))"
   ```

2. **Test with Home Assistant** (if possible)
   - Import the blueprint
   - Create a test automation
   - Verify it works as expected
   - Check for errors in logs

3. **Check documentation**
   - Ensure README is updated
   - Update CHANGELOG.md
   - Update examples if needed
   - Check for broken links

### What We're Looking For

#### High Priority
- Bug fixes
- Performance improvements
- Documentation improvements
- Example configurations
- Compatibility with more HVAC systems

#### Medium Priority
- Support for more zones (6+)
- Advanced scheduling features
- Energy optimization
- Better error handling
- More granular control options

#### Low Priority
- UI improvements
- Additional languages
- Integration with other blueprints
- Alternative calculation methods

### What We're NOT Looking For

- Breaking changes without discussion
- Features that complicate basic usage
- Platform-specific workarounds
- Proprietary integrations
- Overly complex configurations

## Development Setup

### Prerequisites
- Git
- Text editor (VS Code recommended)
- Home Assistant test instance (optional but recommended)
- Python 3 (for YAML validation)

### Recommended Tools
- **VS Code** with extensions:
  - YAML by Red Hat
  - Home Assistant Config Helper
  - Jinja2 Snippet Kit
- **yamllint** for YAML validation
- **Home Assistant Dev Container** for testing

### Local Testing

1. **Clone the repository**
   ```bash
   git clone https://github.com/Chester929/ha_custom_heat_zone_manager.git
   ```

2. **Make changes**
   - Edit `heat_zone_manager.yaml`

3. **Validate**
   ```bash
   yamllint heat_zone_manager.yaml
   ```

4. **Test in Home Assistant**
   - Copy to `<config>/blueprints/automation/chester929/`
   - Reload automations
   - Create test automation
   - Monitor logs and behavior

## Documentation

When contributing, update these files as needed:

- **README.md** - Main documentation, features, examples
- **INSTALLATION.md** - Installation instructions
- **TROUBLESHOOTING.md** - Common issues and solutions
- **CHANGELOG.md** - Version history and changes
- **examples/** - Example configurations

## Release Process

(For maintainers)

1. Update version in blueprint
2. Update CHANGELOG.md
3. Create release notes
4. Tag release in Git
5. Publish to GitHub
6. Announce in community forums

## Community Guidelines

- Be respectful and welcoming
- Help others when you can
- Share your experiences and configurations
- Provide constructive feedback

## Questions?

- Open a [Discussion](https://github.com/Chester929/ha_custom_heat_zone_manager/discussions)
- Join the conversation in existing issues
- Check the [Troubleshooting Guide](TROUBLESHOOTING.md)

## Recognition

Contributors will be:
- Listed in release notes
- Credited in CHANGELOG.md
- Mentioned in the README (for significant contributions)

Thank you for contributing to make floor heating management better for everyone! 🎉
