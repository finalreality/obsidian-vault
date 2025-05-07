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
		- vk::SurfaceFormatKHR
			- vk::Format::eR8G8B8A8Srgb

## vk::Device
- getSwapchainImagesKHR
- vk::SwapchainKHR
- createSwapchainKHR
	- vk::SwapchainCreateInfoKHR
		- setClipped
		- setCompositeAlpha
			- vk::CompositeAlphaFlagBitsKHR::eOpaque
		- setImageExtent
			- vk::Extent2D
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
			- vk::SurfaceTransformFlagBitsKHR
		- setSurface
			- vk::SurfaceKHR
		- setImageSharingMode
			- vk::SharingMode::eExclusive
			- vk::SharingMode::eConcurrent
		- setQueueFamilyIndices
	- vk::SwapchainKHR
- createImageView
	- vk::ImageViewCreateInfo
		- setImage
			- vk::Image
		- setFormat
			- vk::SurfaceFormatKHR
		- setViewType
			- vk::ImageViewType::e2D
		- setSubresourceRange
			- vk::ImageSubresourceRange
				- setBaseArrayLayer
				- setBaseMipLevel
				- setLayerCount
				- setLevelCount
				- setAspectMask
					- vk::ImageAspectFlagBits::eColor
		- setComponents
			- vk::ComponentMapping{}