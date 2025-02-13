[Back to Index](/README.md)

# Building an SDK for LIMP
LIMP currently only has an Angular SDK. We are working with other developers to provide React, React Native, Java and Swift SDKs. However, if you are in need of creating your SDK for any reason, here are the things you need to know:
1. You need to start with a websocket client, connecting to `ws[s]://IP_OR_HOST[:PORT]/ws`.
2. LIMPd would respond with the following:
```json
{ "status": 200, "msg": "Connection established." }
```
3. Your calls to LIMP should have the following structure (Typescript-annotated interface):
```typescript
// [DOC] Query, QueryStep interfaces are strict-typed list-based, object-based accordingly. The interfaces assure that only LIMP Query structure is passed as call query. 
interface QueryStep {
	$search?: string;
	$sort?: {
		[attr: string]: 1 | -1;
	};
	$skip?: number;
	$limit?: number;
	$extn?: false | Array<string>;
	$attrs?: Array<string>;
	$group?: Array<{
		by: string;
		count: number;
	}>;
	[attr: string]: {
		$not: any;
	} | {
		$eq: any;
	} | {
		$gt: number | string;
	} | {
		$gte: number | string;
	} | {
		$lt: number | string;
	} | {
		$lte: number | string;
	} | {
		$bet: [number, number] | [string, string];
	} | {
		$all: Array<any>;
	} | {
		$in: Array<any>;
	} | {
		$attrs: Array<string>;
	} | {
		$skip: false | Array<string>;
	} | Query | string | { [attr: string]: 1 | -1; } | number | false | Array<string>;
}
interface Query extends Array<QueryStep> {}
interface CallArgs {
	call_id?: string; // [DOC] A unique token to distinguish which responses from LIMPd belong to which calls. This is auto-generated by the SDK.
	endpoint?: string; // [DOC] The endpoint you are calling, it's in the form of 'module/method'.
	sid?: string; // [DOC] The session ID. In usual cases this value is supplied internally by the SDK.
	token?: string; // [DOC] The session token. In usual cases this value is supplied internally by the SDK.
	query?: Query; // [DOC] The call query.
	doc?: { // [DOC] The doc object is the raw values you are passing to LIMP app. It should comply with the module `attrs` you are calling.
		[attr: string]: any;
	};
}
```
4. The call should be tokenised using `JWT` standard with the following header, using the session token, or `ANON_TOKEN` if you have not yet been authenticated:
```typescript
interface { alg: 'HS256', typ: 'JWT' }
```
5. To authenticate the user for the current session you need to make the following call:
```typescript
interface {
	call_id: string;
	endpoint: 'session/auth';
	sid: 'f00000000000000000000012';
	doc: { [key: 'username' | 'email' | 'phone']: string; hash: string; }
}
/*
[DOC] You can get the hash of the auth method of choice from 'username', 'email', or 'phone' by generating the JWT of the following list for authHashLevel=5.0:
{
	hash: [authVar, authVal, password]
}
or following list for authHashLevel=5.6:
{
	hash: [authVar, authVal, password, anonToken]
}
signed using 'password' value
*/
```
6. To re-authenticate the user from the cached credentials, in a new session, you can make the following call:
```typescript
interface {
	call_id: string;
	endpoint: 'session/reauth';
	sid: 'f00000000000000000000012';
	query: [ { _id: string; hash: string; } ];
}
/*
[DOC] You can get the hash to reauth method by generating the JWT of the following object:
{
	token: cachedToken
}
signed using cached token value
*/
```
7. Files can only be pushed as part of the `doc` object in the call if you are using the special call to `file/upload`. Right before sending the call, attempt to detect any `FileList` or matching types in the call `doc` object. If any are found, iterate over all the files in the `FileList`, read each as `ByteArray` object and slice it into slices matching the default or user-set SDK `fileChunkSize` attr. Each slice should be sent to LIMPd in the form of:
```typescript
interface {
	attr: string;
	index: number; // Index of the file in the FileList.
	chunk: number; // Index of the chunk of the current file.
	total: number; // Total number of chunks to be sent.
	file: { // Attr file is not user-defined. This should always be the name of the attr.
		name: string; // File name.
		size: number; // File size.
		type: string; // File mime-type.
		lastModified: number; // File disk lastModified value.
		content: string; // The byteArray slice joined with ',' commas e.g. byteArraySlice.join(',')
	}
}
```
The previous `doc` object should be then sent to special endpoint `file/upload` which would parse the file, confirm it's part of an ongoing file upload process and respond with `Chunk accepted` message for all the chunks except the last which would be `Last Chunk accepted`. Once you receive confirmation from LIMP app that all files in all `FileList` objects you detected earlier are uploaded, you then can send the original call. [LIMP SDK for Angular](https://github.com/masaar/ng-limp) has very clear workflow on handling `FileList`s and parse them with the explained method.

## Methods
LIMP SDKs should all look-alike. This means that when you are working on developing new SDK for LIMP you need to make sure it behaves like original authors maintained SDKs:
1. [LIMP SDK for Angular](https://github.com/masaar/ng-limp).
2. [LIMP SDK for Nativescript](https://github.com/masaar/ns-limp).

LIMP SDKs depend on [ReactiveX](http://reactivex.io/) for exposing the SDKs methods as well as for internal use. The SDK you are planning to work on should leverage the usage of ReactiveX in order to unify the experience of using LIMP SDKs across different environments.