<script lang='ts'>
    import '../node_modules/@smui-extra/autocomplete/bare.css';
    import Autocomplete from '@smui-extra/autocomplete';
    import { Text } from '@smui/list';
    import CircularProgress from '@smui/circular-progress';

    import { getContext, onDestroy } from 'svelte';

    export let field;
    export let label;
    export let validation;
    export let dataSourceType;
    export let searchEvent;
    export let apiUrl;
    export let queryParam;
    export let staticOptions;
    export let dataProvider;
    export let loading;
    export let results;
    export let labelFieldName;
    export let valueFieldName;
    export let debug;
    export let allowArbitrary;

    let resultsPromise;
    let loadingResolver;

    const { styleable } = getContext('sdk');
    const component = getContext('component');
    const formContext = getContext('form');
    const formStepContext = getContext('form-step');
    const fieldGroupContext = getContext('field-group');

    let fieldState;
    let fieldApi;
    let errors;
    let content;
    let unsubscribe;
    let text = "";
    let selectedItem;
    let initialItemsPromise;

    const formApi = formContext?.formApi;
    const labelPos = fieldGroupContext?.labelPosition || 'above';
    $: formStep = formStepContext ? $formStepContext || 1 : 1;
    $: labelClass = labelPos === 'above' ? '' : `spectrum-FieldLabel--${labelPos}`;


    $: dataContext = {
        text,
        value: selectedItem?.[valueFieldName]
    };

    $: delay = dataSourceType === 'static' ? 0 : 100;

    $: if (formApi && field) {
        const formField = formApi.registerField(
            field,
            'text',
            '', ///TODO: Default value?
            false,
            validation,
            formStep
        );

        unsubscribe = formField?.subscribe(async (fieldValue) => {
            fieldState = fieldValue?.fieldState;
            fieldApi = fieldValue?.fieldApi;

            const value = fieldState?.value;

            if (value && !initialItemsPromise) {
                console.log('Loading initial items');
                initialItemsPromise = await getItems(value);
                const item = (await initialItemsPromise).find((item) => item[valueFieldName] === value);

                if (item) {
                    selectedItem = item;
                }
                else {
                    selectedItem = {
                        [valueFieldName]: value,
                        [labelFieldName]: value
                    };
                }
            }

        });
    };

    $: if (dataSourceType === 'query' || (dataSourceType === 'budibase' && searchEvent)) {
        const parsedLoading = loading?.toLowerCase() === 'true' || loading === '1';

        if (parsedLoading && !loadingResolver) {
            console.log('Loading new results from query');
            resultsPromise = new Promise((resolve) => {
                loadingResolver = resolve;
            });
        }
        else if (!parsedLoading && loadingResolver) {
            console.log('Got new results from query');

            if (!results) {
                results = []
            }

            if (typeof results === 'string') {
                results = JSON.parse(results);
            }

            if (debug && dataSourceType === 'query') {
                console.log(`Raw results were ${results}`);
            }

            const parsedResults = dataSourceType === 'query' ? results : dataProvider?.rows;
            loadingResolver(parsedResults);
            loadingResolver = null;
        }
    };

    $: if (dataSourceType === 'static') {
        labelFieldName = 'label';
        valueFieldName = 'value';
    };

    onDestroy(() => {
        fieldApi?.deregister();
        unsubscribe?.();
    });

    async function searchItems(keyword: string) {
        const items = await getItems(keyword);

        return items.filter((item) => item[labelFieldName].toLowerCase().includes(keyword));
    }

    async function getItems(keyword: string) {
        const items = await _getItems(keyword);

        if (allowArbitrary) {
            const arbitraryItem = {
                [labelFieldName]: keyword,
                [valueFieldName]: keyword
            }

            return [...items, arbitraryItem];
        }

        return items;
    }

    async function _getItems(keyword: string) {
        switch (dataSourceType) {
            case 'budibase':
                if (!searchEvent) {
                    return dataProvider?.rows;
                }
            case 'query':
                if (searchEvent) {
                    searchEvent({keyword});
                }
                await new Promise((resolve) => setTimeout(resolve, 100));
                return await resultsPromise
            case 'static':
                return staticOptions;
            case 'api':
                const url = new URL(apiUrl);
                url.searchParams.set(queryParam, keyword);
                const response = await fetch(url);
                return await response.json();
        }
    }

    function changeHandler(v) {
        selectedItem = v;

        if (selectedItem) {
            fieldApi?.setValue(selectedItem[valueFieldName]);
        }
    }

    function getOptionLabel(option) {
        if (!option) {
            return;
        }

        if (!labelFieldName) {
            return typeof option === 'string' ? option : JSON.stringify(option);
        }

        return option[labelFieldName];
    }

</script>


<div class='spectrum-Form-item' class:above={labelPos === "above"} use:styleable={$component.styles}>
    {#if !formContext}
        <div class='placeholder'>Form components need to be wrapped in a form</div>
    {:else}
        <div class='spectrum-Form-itemField'>
            <Autocomplete
                inputId={fieldState?.fieldId}
                search="{searchItems}"
                bind:value={() => selectedItem, changeHandler}
                bind:text
                showMenuWithNoInput={false}
                {label}
                {getOptionLabel}
                style="width: 100%;"
                textfield$style="width: 100%;"
            >
                {#snippet loading()}
                    <Text style="display: flex; width: 100%; justify-content: center; align-items: center;" >
                        <CircularProgress style="height: 24px; width: 24px;" indeterminate />
                    </Text>
                {/snippet}
            </Autocomplete>
        </div>
        {#if fieldState?.error}
            <div class='error'>{fieldState.error}</div>
        {/if}
    {/if}
</div>
