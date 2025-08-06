### Testing as Design Practice

Tests are more than quality assurance - they're executable specifications, design tools, and confidence builders. Your test suite is the foundation that enables fearless refactoring and evolution.

#### The London School Approach:
Focus on behavior over implementation, using mocks to isolate concerns and verify interactions. Your tests should read like specifications written for both humans and machines.

#### Test-Driven Design Process:
1. **Red**: Write a failing test that expresses desired behavior
2. **Green**: Implement minimal code to make the test pass
3. **Refactor**: Improve design while maintaining passing tests
4. **Reflect**: Update memory-bank/systemPatterns.md with insights

#### Testing Aesthetics:
- **Test Names**: Should read like specification statements
- **Test Structure**: Arrange, Act, Assert with clear boundaries
- **Test Data**: Meaningful examples that illuminate edge cases
- **Mock Strategy**: Precise boundaries between units of concern

#### Quality Sensing Through Tests:
- **Test Brittleness**: When tests break frequently with minor changes
- **Test Clarity**: When test failure messages immediately reveal problems
- **Coverage Confidence**: When test suite genuinely catches regressions
- **Design Feedback**: When testing reveals design improvements

#### Integration with SPARC:
- **Specification Phase**: Transform acceptance criteria into test outlines
- **Pseudocode Phase**: Design testable interfaces and contracts
- **Architecture Phase**: Plan testing strategies and mock boundaries
- **Refinement Phase**: Implement comprehensive test coverage
- **Completion Phase**: Validate end-to-end scenarios