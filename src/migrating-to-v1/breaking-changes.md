## Breaking changes

When it comes to breaking changes, there has been a slight rework for the assertion (`expect*`) cheatcodes that removes previously allowed unintended or confusing behavior.

The biggest change to expect, is that these cheatcodes will now work only for the next call, not at the "test" level anymore. The philosophy behind this is that by making these cheatcodes more strict through this requirement, tests will be more accurate as the cheatcodes will now succeed under more strict conditions.

#### `expectEmit`

`expectEmit` previously allowed you to assert that a log was emitted during the execution of the test. With V1, the following changes have been made:

- it now only works for the _next call_. This means if you used to declare all your expected emits at the top of the test, you may now have to move them to just before the next call you perform. Cheatcode calls do not count. As long as the events are emitted during the next call's context, they will be matched.
- The events have to be _ordered_. That means, if you're trying to match events [A, B, C], you must declare them in this order.
    - Skipping events is possible. Therefore, if a function emits events [A, B, C, D, E] and you want to match [B, D, E], you just need to declare them in that order.

To illustrate, see the following examples:

```solidity

contract Emitter {
    event A(uint256 indexed a);
    event B(uint256 indexed b);
    event C(uint256 indexed c);
    event D(uint256 indexed d);
    event E(uint256 indexed e);

    /// emit() emits events [A, B, C, D, E]
    function emit() external {
        emit A(1);
        emit B(2);
        emit C(3);
        emit D(4);
        emit E(1);
    }
}

contract EmitTest is Test {
    Emitter public emitter;

    event A(uint256 indexed a);
    event B(uint256 indexed b);
    event C(uint256 indexed c);
    event D(uint256 indexed d);
    event E(uint256 indexed e);

    function setUp() public {
        emitter = new Emitter();
    }

    /// CORRECT BEHAVIOR: Declare all your expectEmits just before the next call,
    /// on the test.
    /// emit() emits [A, B, C, D, E], and we're expecting [A, B, C, D, E].
    /// this passes.
    function testExpectEmit() public {
        cheats.expectEmit(true, false, false, true);
        emit A(1);
        cheats.expectEmit(true, false, false, true);
        emit B(2);
        cheats.expectEmit(true, false, false, true);
        emit C(3);
        cheats.expectEmit(true, false, false, true);
        emit D(4);
        cheats.expectEmit(true, false, false, true);
        emit E(5);
        emitter.emit();
    }

    /// CORRECT BEHAVIOR: Declare all your expectEmits just before the next call,
    /// on the test.
    /// emit() emits [A, B, C, D, E], and we're expecting [B, D, E].
    /// this passes.
    function testExpectEmitWindow() public {
        cheats.expectEmit(true, false, false, true);
        emit B(2);
        cheats.expectEmit(true, false, false, true);
        emit D(4);
        cheats.expectEmit(true, false, false, true);
        emit E(5);
        emitter.emit();
    }

    /// INCORRECT BEHAVIOR: Declare all your expectEmits in the wrong order.
    /// emit() emits [A, B, C, D, E], and we're expecting [D, B, E].
    /// this fails, as D is emitted after B, not before.
    function testExpectEmitWindowFailure() public {
        cheats.expectEmit(true, false, false, true);
        emit D(4);
        cheats.expectEmit(true, false, false, true);
        emit B(2);
        cheats.expectEmit(true, false, false, true);
        emit E(5);
        emitter.emit();
    }

    /// CORRECT BEHAVIOR: Declare all your expectEmits in an internal function.
    /// Calling a contract function internally is a JUMPI, not a call,
    /// therefore it's a valid pattern, useful if you have a lot of events to expect.
    /// emit() emits [A, B, C, D, E], and we're expecting [B, D, E].
    /// this passes.
    function testExpectEmitWithInternalFunction() public {
        declareExpectedEvents();
        emitter.emit();
    }

    function declareExpectedEvents() internal {
        cheats.expectEmit(true, false, false, true);
        emit B(2);
        cheats.expectEmit(true, false, false, true);
        emit D(4);
        cheats.expectEmit(true, false, false, true);
        emit E(5);
    }
}
```

As illustrated above, if you'd like to make your tests more brief, you can declare your expected events in an internal function in your test contract to remove visual clutter. This is not an external call, which means they won't count for detecting events.

#### `expectCall`

Just like `expectEmit`, `expectCall` has gotten a similar rework. Before, it worked at the test level, which meant you could declare the calls you expect to see upfront and then start calling external functions. The new behavior is the following one:

- It now only works for the next _call's subcalls_. This means that the call(s) you expect need to happen inside an external call for them to be detected. Calling it at the test level (directly) won't count.
    - This is explained through "depth": the `expectCall` cheatcode is called at a "test-level" depth, so we enforce that the calls we expect to see are made at a "deeper" depth to be able to count them properly. Depth is increased whenever we make an external call. Internal calls do not increase depth.

To illustrate, see the following examples:

```solidity
contract Protocol {
    function doSomething() external {
        // do something...
    }
}

contract Caller {
    Protocol public protocol;

    function foo() public {
        // do something...
    }

    function baz() public {
        // do something...
    }

    function bar() public {
        protocol.doSomething();
        // do something...
    }
}

contract ExpectCallTest is Test {
    Caller public caller;

    function setUp() public {
        caller = new Caller();
    }

    /// CORRECT BEHAVIOR: We expect a call to `doSomething()` in the next call,
    /// So we declare our expected call, and then call `caller.bar()`, which will
    /// call `protocol.doSomething()` internally.
    /// `doSomething()` is nested inside `bar`, so this passes.
    function testExpectCall() public {
        vm.expectCall(address(caller.protocol), abi.encodeCall(caller.protocol.doSomething));
        // This will call bar internally, so this is valid.
        caller.bar();
    }

    /// INCORRECT BEHAVIOR: We expect a call to `doSomething()` in the next call,
    /// So we declare our expected call, and then call `protocol.doSomething()` directly.
    /// `doSomething()` is not nested, but rather called at the test level.
    /// This doesn't satisfy the depth requirement described above,
    /// so this fails.
    function testExpectCallFailure() public {
        vm.expectCall(address(caller.protocol), abi.encodeCall(caller.protocol.doSomething));
        // We're calling doSomething() directly here.
        // This is not inside the next call's subcall tree, but rather a test-level
        // call. This fails.
        caller.protocol.doSomething();
    }

    /// CORRECT BEHAVIOR: Sometimes, we want to directly expect the call we're gonna
    /// call next, but we need to satisfy the depth requirement. In those cases,
    /// this pattern can be used.
    /// Calling exposed_callFoo using `this` will create a new call context which
    /// will increase depth. In its subcall tree, we'll detect `exposed_callFoo`,
    /// which will make the test pass.
    function testExpectCallWithNest() public {
        vm.expectCall(address(caller), abi.encodeWithSelector(caller.foo.selector));
        this.exposed_callFoo();
    }

    function exposed_callFoo() public {
        caller.foo();
    }
}
```

#### `expectRevert`

`expectRevert` currently allows to expect for reverts at the test level. This means that a revert can be declared at the top of the test and as long as a function in the test reverts, the test will be pass. The new behavior is the following:

- `expectRevert` CANNOT be used with other `expect` cheatcodes. It will fail if it detects they're being used at the same time.
    - The reasoning for this is that calls that revert are rolled back, and events and calls therefore never happen. This restrictions helps model real-world behavior.
- `expectRevert` now works only for the next call. If the next call does not revert, the cheatcode will fail and therefore the test will also fail. The depth of the revert does not matter: as long as the next call reverts, either inmediately or deeper into its call subtree, the cheatcode will succeed.

To illustrate, see the following examples:

```solidity

contract Reverter {
    function revertWithMessage(string memory message) public pure {
        require(false, message);
    }
    
    function doNotRevert() public {
        // do something unrelated, do not revert.
    }
}

contract ExpectRevertTest is Test {
    Reverter reverter;

    function setUp() public {
        reverter = new Reverter();
    }

    /// CORRECT BEHAVIOR: We expect `revertWithMessage` to revert,
    /// so we declare a expectRevert and then call the function.
    function testExpectRevertString() public {
        vm.expectRevert("revert");
        reverter.revertWithMessage("revert");
    }

    /// INCORRECT BEHAVIOR: We correctly expect that a function will revert in this
    /// test, but we declare the expected revert before the wrong function,
    /// `doNotRevert()`, which does not revert. The cheatcode therefore fails.
    function testFailRevertNotOnImmediateNextCall() public {
        // expectRevert should only work for the next call. However,
        // we do not inmediately revert, so,
        // we fail.
        vm.expectRevert("revert");
        reverter.doNotRevert();
        reverter.revertWithMessage("revert");
    }
}
```

### Testing internal calls or libraries with `expectCall` and `expectRevert`

With the changes to `expectCall` and `expectRevert`, library maintainers might need to modify tests that expect calls or reverts on library functions that might be marked as internal. As an external call needs to be forced to satisfy the depth requirement, an intermediate contract that exposes an identical API to the function that needs to be called can be used.

See the following example for testing a revert on a library.

```solidity
library MathLib {
    function add(uint a, uint b) internal pure {
        return a + b;
    }
}

// intermediate "mock" contract that calls the library function
// and returns the result.
contract MathLibMock {
    function add(uint a, uint b) external {
        return MathLib.add(a, b);
    }
}

contract MathLibTest is Test {
    MathLibMock public mock;

    function setUp() public {
        mock = new MathLibMock();
    }

    /// INCORRECT BEHAVIOR: MathLib.add will revert due to arithmetic errors,
    /// but as the function is marked as internal, it'll be a test level revert
    /// instead of a call revert that expectRevert is supposed to catch.
    /// The test will fail, and the error will let you know that you have a
    /// "dangling" expectRevert.
    function testRevertOnMathLibWithNoMock() public {
        vm.expectRevert();
        MathLib.add(2 ** 128 - 1, 2 ** 255 - 1);
    }

    /// CORRECT BEHAVIOR: mock.add will revert due to arithmetic errors,
    /// and it will be successfully detected by the `expectRevert` cheatcode.
    function testRevertOnMathLibWithMock() public {
        vm.expectRevert();
        mock.add(2 ** 128 - 1, 2 ** 255 - 1);
    }
}
```