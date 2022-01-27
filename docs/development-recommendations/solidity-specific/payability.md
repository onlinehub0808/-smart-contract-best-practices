Starting from Solidity `0.4.0`, every function that is receiving ether must use `payable` modifier,
otherwise if the transaction has `msg.value > 0` will revert
([except when forced](./recommendations/#remember-that-ether-can-be-forcibly-sent-to-an-account)).

!!! Note
    Something that might not be obvious: The `payable` modifier only applies to calls from *external* contracts. If I call a non-payable function in the payable function in the same contract, the non-payable function won't fail, though `msg.value` is still set.
