# Microsoft SharePoint 2013 API - Editing Read-Only Fields
Typically, an item's fields (columns) can be created and edited via the web GUI in a list/library's settings. This can also be done with the API, which gives slightly more editing control over the fields, in this case, allowing read-only fields to be modified.

## Noting the fields of a list's items
Using jQuery for the AJAX calls, the available fields on an item can be retrieved:
```
$.ajax({
	url: _spPageContextInfo.webAbsoluteUrl + "/_api/web/lists/getByTitle('List Name Here')/fields",
	type: "GET",
	contentType: "application/json; odata=verbose",
	headers: { 
		"Accept": "application/json; odata=verbose",
		"X-RequestDigest": $("#__REQUESTDIGEST").val(),
		"X-HTTP-Method": "GET"
	},
	success: function(res) {
		console.log(res);
	},
	error: function(res) {
		console.log(res);
	}
});
```
* _spPageContextInfo.webAbsoluteUrl gives the current web URL site location, you can type it in a developer console to immediately see what it returns
* There may be unintended effects of capitalizing "lists" and "fields" when using getByTitle or the lists(guid'...')/fields(guid'...') methods

The return response object, once expanded out, should contain an array of results, each index of the array describing a particular field of an item of the list being observed.

For this example, we can say we want to complete a **Survey** but set the _Created By_ and _Modified By_ fields to be a test user account, making this an anonymous survey.
> Note the implications this may have, as now there is no trace of who the original user is from this perspective.

At this point, if we try to make a POST call to create an item in the list while changing the _Created By_ and _Modified By_ fields, it would create under the currently logged in user's name.

## Changing read-only to false
In the initial call earlier to the item's fields, _Created By_ and _Modified By_ could be seen in the objects in the results array, but they both had a property of _ReadOnlyField: true_.

The API endpoint `/_api/web/lists/getByTitle('List Name Here')/fields/getByTitle('Field Name Here')/readOnlyField` is not very helpful, performing either a GET or a POST on it merely returns the status of that field.

To change the fields here, we use the following call:
```
$.ajax({
	url: _spPageContextInfo.webAbsoluteUrl + "/_api/Web/lists/getByTitle('List Name Here')/fields/getByTitle('Created By')",
	type: "POST",
	contentType: "application/json;odata=verbose",
	data:  "{ '__metadata': { 'type': 'SP.Field' }, 'ReadOnlyField': false}",
	headers: { 
		"Accept": "application/json; odata=verbose",
		"X-RequestDigest": $("#__REQUESTDIGEST").val(),
		"X-HTTP-Method": "MERGE"
	},
	success: function(res) {
		console.log(res);
	},
	error: function(res) {
		console.log(res);
	}
});
```
* Note that we are using method POST, and within the headers, _X-HTTP-Method_ is set to MERGE. While APIs often accept GET and POST requests as the standard, other methods such as PUT and DELETE may not always be allowed. Here, MERGE is specifying a specific type of POST request specific to SharePoint. MERGE in this case is used for making updates to these fields.

There is a new key:value pair now since we are making a POST that requires a JSON payload. The metadata field is standard in all payloads of SharePoint objects, and fields have type _SP.Field_. Then, we want to set _ReadOnlyField_ to false.

If you were to try `/_api/web/lists/getByTitle('List Name Here')/fields/getByTitle('Field Name Here')/readOnlyField` again, the property should now say false.

## Sending a payload 
```
var payload = {
	"__metadata": { "type": "SP.Data.InternalListNameHereListItem" },
	"Title": 'My Item That Was Not Created By Me',
	"AuthorId": TestUserAccountID,
	"EditorId": TestUserAccountID
};
```
* InternalListNameHere is the internal list name of the list being worked with. A quick GET to `/_api/Web/lists/getByTitle('List Name Here')` should reveal it. For example, if the list was called "Responses", the internal name might be "Responses as well, so the metadata type property becomes _SP.Data.ResponsesListItem_. "Survey Responses" might become _SP.Data.Survey_x0020_ResListItem_ depending on how much of the name is cut off and treated internally. It has been noted that there may be issues if the very first letter of the internal list name is not capitalized when setting the _SP.Data. ... ListItem_ property.
* TestUserAccountID is the ID of the account that can be seen making all the anonymous responses in this case. You could briefly make an item under that particular account in a list and call on `/_api/web/lists/getByTitle('List Name Here')/items` to find that item, and expand its properties to reveal the _Created By_ property which would give the ID.


Send the payload with the following (or bind it to a list's Save button so that in this case, it creates the item on save (where a practical example of completing a survey might also get all the values out of their fields and put it in the payload)):
```
$.ajax({
	url: _spPageContextInfo.webAbsoluteUrl + "/_api/web/lists/getByTitle('List Name Here')/items",
	type: "POST",
	contentType: "application/json; odata=verbose",
	data: JSON.stringify(payload),
	headers: { 
		"Accept": "application/json; odata=verbose",
		"X-RequestDigest": $("#__REQUESTDIGEST").val(),
		"X-HTTP-Method": "POST"
	},
	success: function(res) {
		console.log(res);
	},
	error: function(res) {
		console.log(res);
	}
});
```

If everything went well, the new item should be created and modified by the test account.



While this may have some application with anonymous surveys, **_if a user edits the list, the Created By and Modified By properties are still not read-only, and thus can be changed to anyone_**.
