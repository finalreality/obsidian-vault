---
mindmap-plugin: basic
---

# Vulkan

## vk::createInstance
- vk::InstanceCreateInfo
	- setPApplicationInfo
		- vk::ApplicationInfo
			- sType
			- setApiVersion
				- VK_API_VERSION_1_0
			- setPApplicationName

## vk::Instance
- enumeratePhysicalDevices

## vk::PhysicalDevice
- createDevice
	- vk::DeviceCreateInfo
		- setQueueCreateInfos
	- vk::DeviceQueueCreateInfo
		- setPQueuePriorities
		- setQueueCount
		- setQueueFamilyIndex
- getProperties
- getQueueFamilyProperties

## vk::Device