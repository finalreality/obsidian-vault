---
mindmap-plugin: basic
---

# Vulkan

## vk::createInstance
- vk::InstanceCreateInfo
	- setPApplicationInfo
		- vk::ApplicationInfo
			- sType
				- vk::StructureType::eApplicationInfo
			- setApiVersion
				- VK_API_VERSION_1_3
			- setPApplicationName
	- setPEnabledExtensionNames
	- setPpEnabledLayerNames
		- std::vector<const char*> layers

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

## vk::SwapchainKHR

## vk::SurfaceKHR