<script lang="ts">
  import type { FetchBalanceResult } from '@wagmi/core';
  import { t } from 'svelte-i18n';
  import { formatUnits, parseUnits } from 'viem';

  import { FlatAlert } from '$components/Alert';
  import { InputBox } from '$components/InputBox';
  import { LoadingText } from '$components/LoadingText';
  import { warningToast } from '$components/NotificationToast';
  import { checkBalanceToBridge, getMaxAmountToBridge } from '$libs/bridge';
  import { InsufficientAllowanceError, InsufficientBalanceError, RevertedWithFailedError } from '$libs/error';
  import { getBalance as getTokenBalance } from '$libs/token';
  import { debounce } from '$libs/util/debounce';
  import { truncateString } from '$libs/util/truncateString';
  import { uid } from '$libs/util/uid';
  import { account } from '$stores/account';
  import { network } from '$stores/network';

  import {
    computingBalance,
    destNetwork,
    enteredAmount,
    errorComputingBalance,
    insufficientAllowance,
    insufficientBalance,
    processingFee,
    recipientAddress,
    selectedToken,
    tokenBalance,
  } from './state';

  let inputId = `input-${uid()}`;
  let inputBox: InputBox;
  let computingMaxAmount = false;

  // Public API
  export function clearAmount() {
    inputBox.clear();
    $enteredAmount = BigInt(0);
  }

  export async function validateAmount(token = $selectedToken, fee = $processingFee) {
    $insufficientBalance = false;
    $insufficientAllowance = false;

    const to = $recipientAddress || $account?.address;

    // We need all these guys to validate
    if (
      !to ||
      !token ||
      !$network ||
      !$destNetwork ||
      !$tokenBalance ||
      $enteredAmount === BigInt(0) // no need to check if the amount is 0
    )
      return;

    try {
      await checkBalanceToBridge({
        to,
        token,
        amount: $enteredAmount,
        processingFee: fee,
        balance: $tokenBalance.value,
        srcChainId: $network.id,
        destChainId: $destNetwork.id,
      });
    } catch (err) {
      console.error(err);

      switch (true) {
        case err instanceof InsufficientBalanceError:
          $insufficientBalance = true;
          break;
        case err instanceof InsufficientAllowanceError:
          $insufficientAllowance = true;
          break;
        case err instanceof RevertedWithFailedError:
          warningToast($t('messages.network.rejected'));
          break;
      }
    }
  }

  export async function updateBalance(
    token = $selectedToken,
    userAddress = $account?.address,
    srcChainId = $network?.id,
    destChainId = $destNetwork?.id,
  ) {
    if (!token || !srcChainId || !userAddress) return;

    $computingBalance = true;
    $errorComputingBalance = false;

    try {
      $tokenBalance = await getTokenBalance({
        token,
        srcChainId,
        destChainId,
        userAddress,
      });
    } catch (err) {
      console.error(err);
      $errorComputingBalance = true;
    } finally {
      $computingBalance = false;
    }
  }

  // We want to debounce this function for input events.
  // Could happen as the user enters an amount
  const debouncedValidateAmount = debounce(validateAmount, 300);

  function renderBalance(balance: Maybe<FetchBalanceResult>) {
    if (!balance) return '0.00';

    return `${truncateString(balance.formatted, 6)} ${balance.symbol}`;
  }

  // Will trigger on input events. We update the entered amount
  // and check it's validity
  function inputAmount(event: Event) {
    if (!$selectedToken) return;

    const { value } = event.target as HTMLInputElement;
    $enteredAmount = parseUnits(value, $selectedToken.decimals);

    debouncedValidateAmount();
  }

  // "MAX" button handler
  async function useMaxAmount() {
    // We cannot calculate the max amount without these guys
    if (!$selectedToken || !$network || !$destNetwork || !$tokenBalance || !$account?.address) return;

    computingMaxAmount = true;

    try {
      const maxAmount = await getMaxAmountToBridge({
        to: $recipientAddress || $account.address,
        token: $selectedToken,
        balance: $tokenBalance.value,
        processingFee: $processingFee,
        srcChainId: $network.id,
        destChainId: $destNetwork.id,
        amount: BigInt(1), // whatever amount to estimate the cost
      });

      // Update UI
      // Note: triggering the event manually does not always work, specially
      // in other browsers (looking at you, Safari!!)
      inputBox.setValue(formatUnits(maxAmount, $selectedToken.decimals));

      // Update state
      $enteredAmount = maxAmount;

      // Check validity
      validateAmount();
    } catch (err) {
      console.error(err);
      warningToast($t('amount_input.button.failed_max'));
    } finally {
      computingMaxAmount = false;
    }
  }

  $: updateBalance($selectedToken, $account?.address, $network?.id, $destNetwork?.id);

  $: validateAmount($selectedToken, $processingFee);
</script>

<div class="AmountInput f-col space-y-2">
  <div class="f-between-center text-secondary-content">
    <label class="body-regular" for={inputId}>{$t('amount_input.label')}</label>
    <div class="body-small-regular">
      <span>{$t('amount_input.balance')}:</span>
      <span>
        {#if $computingBalance}
          <LoadingText mask="0.0000" />
          <LoadingText mask="XXX" />
        {:else}
          {renderBalance($tokenBalance)}
        {/if}
        <!-- TODO: we know when there was an error computing. Show it -->
      </span>
    </div>
  </div>

  <div class="relative">
    <div class="relative f-items-center">
      <InputBox
        id={inputId}
        type="number"
        placeholder="0.01"
        min="0"
        loading={computingMaxAmount}
        error={$insufficientBalance}
        on:input={inputAmount}
        bind:this={inputBox}
        class="w-full input-box outline-none py-6 pr-16 px-[26px] title-subsection-bold placeholder:text-tertiary-content" />
      <!-- TODO: talk to Jane about the MAX button and its styling -->
      <button
        class="absolute right-6 uppercase hover:font-bold"
        disabled={!$selectedToken || !$network || computingMaxAmount}
        on:click={useMaxAmount}>
        {$t('amount_input.button.max')}
      </button>
    </div>

    {#if $insufficientBalance}
      <FlatAlert type="error" message={$t('amount_input.error.insufficient_balance')} class="absolute bottom-[-26px]" />
    {/if}

    {#if $insufficientAllowance}
      <FlatAlert
        type="warning"
        message={$t('amount_input.error.insufficient_allowance')}
        class="absolute bottom-[-26px]" />
    {/if}
  </div>
</div>
