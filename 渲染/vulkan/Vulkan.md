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
- getSurfaceCapabilitiesKHR
	- vk::SurfaceTransformFlagBitsKHR

## vk::Device
- getSwapchainImagesKHR
- createSwapchainKHR
	- vk::SwapchainCreateInfoKHR
		- setClipped
		- setCompositeAlpha
			- vk::CompositeAlphaFlagBitsKHR::eOpaque
		- setImageExtent
		- setImageColorSpace
			- vk::ColorSpaceKHR::eSrgbNonlinear
		- setImageFormat
			- vk::Format::eR8G8B8A8Srgb
		- setImageUsage
			- vk::ImageUsageFlagBits::eColorAttachment
		- setMinImageCount
		- setImageArrayLayers
		- setPresentMode
			- vk::PresentModeKHR::eFifo
		- setPreTransform
		- setSurface
			- vk::SurfaceKHR
		- setImageSharingMode
			- vk::SharingMode::eExclusive
			- vk::SharingMode::eConcurrent
		- setQueueFamilyIndices

## vk::SwapchainKHR