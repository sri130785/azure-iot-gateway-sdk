# Module Requirements

##Overview
This is the documentation for a module that communicates with other modules through a message broker. 
Every module needs to implement the same interface. However, the implementation is module-specific.

##References

##Exposed API
```C
#ifndef MODULE_H
#define MODULE_H

/** @brief Represents a handle to a particular module.*/
typedef struct MODULE_TAG MODULE;
typedef void* MODULE_HANDLE;
typedef struct MODULE_API_TAG MODULE_API;

#include "azure_c_shared_utility/macro_utils.h"
#include "broker.h"
#include "message.h"


#ifdef __cplusplus
extern "C"
{
#endif
    /** @brief	Structure used to represent/abstract the idea of a module.  May
    *			contain Hamdle/FxnPtrs or an interface ptr or some unforseen
    *			representation.
    */
    struct MODULE_TAG
    {
        /** @brief Struct containing function pointers */
        const MODULE_API* module_api;
        /** @brief HANDLE for module. */
        MODULE_HANDLE module_handle;
    };

    /** @brief		Creates a module using the specified configuration connecting
    *				to the specified message broker.
    *
    *	@details	This function is to be implemented by the module creator.
    *
    *	@param		broker		The #BROKER_HANDLE onto which this module
    *								will connect.
    *	@param		configuration	A pointer to the user-defined configuration 
    *								structure for this module.
    *
    *	@return		A non-NULL #MODULE_HANDLE upon success, or @c NULL upon 
    *			failure.
    */
    typedef MODULE_HANDLE(*pfModule_Create)(BROKER_HANDLE broker, const void* configuration);

    /** @brief		Disposes of the resources allocated by/for this module.
    *
    *	@details	This function is to be implemented by the module creator.
    *
    *	@param		moduleHandle	The #MODULE_HANDLE of the module to be destroyed.
    */
    typedef void(*pfModule_Destroy)(MODULE_HANDLE moduleHandle);

    /** @brief		The module's callback function that is called upon message 
    *				receipt.
    *
    *	@details	This function is to be implemented by the module creator.
    *
    *	@param		moduleHandle	The #MODULE_HANDLE of the module receiving the
    *								message.
    *	@param		messageHandle	The #MESSAGE_HANDLE of the message being sent
    *								to the module.
    */
    typedef void(*pfModule_Receive)(MODULE_HANDLE moduleHandle, MESSAGE_HANDLE messageHandle);

	/** @brief		The module may start activity.  This is called when the broker
	*				is guaranteed to be ready to accept messages from this module.
	*
	*	@details	This function may be implemented by the module creator.
	*
	*	@param		moduleHandle	The #MODULE_HANDLE of the module receiving the
	*								message.
	*/
	typedef void(*pfModule_Start)(MODULE_HANDLE moduleHandle);

    /** @brief	Module API version.
    */
    typdef enum MODULE_API_VERSION_TAG
    {
        MODULE_API_VERSION_1
    } MODULE_API_VERSION;

    /** @brief	Module APIs corresponding to Version 1.0 of the gateway. This
    *           contains function pointers to the module specific 
    *           implementations of the interface.
    */
    struct MODULE_API_1
    {
        /** @brief Function pointer to the #Module_Create function. */
        pfModule_Create Module_Create;

        /** @brief Function pointer to the #Module_Destroy function. */
        pfModule_Destroy Module_Destroy;

        /** @brief Function pointer to the #Module_Receive function. */
        pfModule_Receive Module_Receive;

		/** @brief Function pointer to the #Module_Start function (optional). */
		pfModule_Start Module_Start;
    };

    /** @brief	Union of all Module API versions.
    */
    typedef union MODULE_API_UNION_TAG
    {
        /** @brief reference to 1.0 gateway version. */
        struct MODULE_API_1 api1;
    } MODULE_API_UNION;

    /** @brief	Structure returned by ::Module_GetApi containing the API 
    *           version and module specific implementation of the interface. By
    *           convention, api_version is always an enum and always the first
    *            element of this structure.
    */
    struct MODULE_API_TAG
    {
        /** @brief The version of the MODULES_API, shall always be the first
        *          element of this structure
        */
        MODULE_API_VERSION api_version;
        
        /** @brief The module API.  The elements of the union correspond 1:1 
        *          with the elements of the MODULE_API_VERSION enum.
        */
        MODULE_API_UNION api;
    };

    /** @brief	This is the only function exported by a module. Using the
    *			exported function, the caller learns the functions for the 
    *			particular module.
    *	@param	gateway_api_version	The current API version of the gateway.
    *	@return NULL in failure, MODULE_API* pointer on success.
    */
    typedef MODULE_API* (*pfModule_GetApi)(const MODULE_API_VERSION gateway_api_version);

#ifdef _WIN32
#define MODULE_EXPORT __declspec(dllexport)
#else
#define MODULE_EXPORT
#endif // _WIN32

/** @brief	Forms a new name that is by convention the name of the (only) exported 
*			function from a static module library.
*/
#define MODULE_STATIC_GETAPI(MODULE_NAME) C2(Module_GetApi_, MODULE_NAME)

/** @brief	This is the only function exported by a module under a "by
*			convention" name. Using the exported function, the caller learns
*			the functions for the particular module.
*/
MODULE_EXPORT  MODULE_API* Module_GetApi(const MODULE_API_VERSION gateway_api_version);

#ifdef __cplusplus
}
#endif

#endif // MODULE_H
```
## Module_GetApi
```c
typedef MODULE_API* (*pfModule_GetApi)(const MODULE_API_VERSION gateway_api_version);
```

This function is to be implemented by the module creator. It will return a 
pointer to a MODULES_API, or NULL if not successful.

The `MODULE_API` structure shall contain the `api_version` and shall fill in the corresponding api structure in the `api` union. The `gateway_api_version` is passed to the module so a module may decide how to fill in the `MODULE_API` structure.

##Module_Create
```C
static MODULE_HANDLE Module_Create(BROKER_HANDLE broker, const void* configuration);
```
This function is to be implemented by the module creator. It receives the message broker 
to which the module will publish its messages, and a pointer provided by the user
containing configuration information (usually information needed by the module to start).

The function returns a non-`NULL` value when it succeeds, known as the module handle. 
If the function fails internally, it should return `NULL`.

##Module_Destroy
```C
static void Module_Destroy(MODULE_HANDLE moduleHandle);
```
This function is to be implemented by the module creator. The function receives a previously
created handle by `Module_Create`. If parameter `moduleHandle` is `NULL` the function should 
return without taking any action. Otherwise, the function should free all system resources
allocated by the module.

##Module_Receive
```C
static void Module_Receive(MODULE_HANDLE moduleHandle, MESSAGE_HANDLE messageHandle);
```
This function is to be implemented by the module creator. This function is called by the
framework. This function is not called re-entrant. This function shouldn't assume it is 
called from the same thread.

## Module_Start
```c
static void Module_Start(MODULE_HANDLE moduleHandle);
```

This function may be implemented by the module creator.  It is allowed to be `NULL` in the `MODULE_API` structure. If defined, this function is called by the framework when the message broker is guaranteed to be ready to accept messages from the module.
