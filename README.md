# Auto-Complete Plugin

# Description
Budibase Auto-Complete plugin.

Find out more about [Budibase](https://github.com/Budibase/budibase).

This plugin is based on https://github.com/pstanoev/simple-svelte-autocomplete

This brings in a dynamic auto-complete field to complement the existing Budibase fields.

## Instructions

Install the plugin and use is as you would any other string form field. The auto complete data source support 4 different modes:
 - Budibase Datasource (Supports the [Asynchronous Flow](#Asynchronous%20Flow))
 - Custom Query (Requires the [Asynchronous Flow](#Asynchronous%20Flow))
 - Direct API (To a public API directly from the client)
 - Static Values (Similar to Budibase's option picker)

In all cases, the plugin expects an array of objects. You can configure which field of the object is used as the label and which one is used for the value.

If using Custom Query, your query will need to support searching by the value field (probably selected item ID). Upon item select, an additional query is performed.

If you want to support arbitrary text in addition to selections, the plugin exposes a **text** binding.

### Asynchronous Flow
Due to limitation on how events and data are processed in Budibase, an asynchronous flow using flags and state variables. To use the asynchronous flow, the following steps should be followed:
 - Bind a *loading* state variable to the **Loading** setting.
 - For the *Custom Query* mode, bind a *results* state variable to the **Results** setting.
 - Add the following actions to the **Search Event**:
	 1. Set the *loading* state variable to **"true"**
	 2. ...Some query actions *(The current input text is exposed as a binding)*
	 3. For the *Custom Query* mode, set the *results* state variable to the query results
	 4. Set the *loading* state variable to **"false"**
