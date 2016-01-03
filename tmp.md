commit d15b717a80686315f53fc95675b2810f87db9358
Author: Mike Pennisi <mike@mikepennisi.com>
Date:   Sat Jan 2 15:23:03 2016 -0500

    Improve W120
    
    Warning 120 (message "You might be leaking a variable ({a}) here.") is
    triggered when a binding statement contains an assignment expression whose left
    hand side is an identifier. For instance, the assignment to `x` triggers the
    warning in each of the following three statements:
    
        var v = x = null;
        let l = x = null;
        const c = x = null;
    
    In these contexts, it is possible that the author mistakenly believes that a
    new binding for the `x` identifier is being created. Warning 120 was presumably
    implemented to help developers avoid unintentionally creating global
    bindings.
    
    As implemented, JSHint does not consider whether the binding in question has
    already been created, so the following code also triggers the warning (despite
    being safe):
    
        var y;
        var v2 = y = null;
    
    JSHint maintains environment records for each scope, so this problem could be
    resolved by referencing the containing scope(s). Such a solution would not
    address a more fundamental problem: the warning is mostly duplicative of W117
    ("{a} is not defined."). Users are already alerted of the "leakage" when the
    `undef` option is set. JSHint already operates as expected for the following
    input, issuing W117 for line 2 but *not* for line 4:
    
        // jshint undef: true, -W120
        var v = x = null;
        var y;
        var v2 = y = null;
    
    W117 is preferable for two reasons:
    
    1. it is much more robust, as it covers a wide array of additional instances
       where global bindings are being created
    2. it is controlled by a dedicated linting option that is human-readable and
       well documented
    
    Although this makes the warning a good candidate for outright removal, the
    original intention suggests a special case that W120 is uniquely positioned to
    address: re-assignment of bindings in a "higher" scope. For example:
    
        let v1 = 0;
        {
          let v2 = v1 = 1;
        }
    
    Just as in previous examples, the author of this code may have mistakenly
    assumed that the second `let` statement created a new `v1` binding and that
    each instance of the `v1` identifier referenced a unique block-scoped binding.
    This is incorrect for the same reasons, and the mistake has similar
    implications for program correctness.
    
    However, unlike previous examples, this mistake would *not* be
    identified by `undef`/W117 because `v1` is defined.
    
    Refactoring W120 to address this concern is backwards compatabile with
    previous versions of JSHint because this is a special case of the
    previously-implemented behavior (the overall effect is a relaxing of
    linting constraints).
    
    Re-write the conditions that trigger warning W120 to be limited to only
    those cases where the nested assignment expression may be misinterpreted
    as shadowing a binding in a higher scope. Update the warning message to
    describe the more specific issue.

 src/jshint.js               | 17 ++++++++++-------
 src/messages.js             |  2 +-
 src/scope-manager.js        |  4 ++++
 tests/unit/core.js          | 14 ++++----------
 tests/unit/fixtures/leak.js |  3 +++
 5 files changed, 22 insertions(+), 18 deletions(-)
