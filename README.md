# [zemscripten](https://github.com/zig-gamedev/zemscripten)

Zig build package and bindings for an externally configured [Emscripten](https://emscripten.org) toolchain.

## How to use it

This integration branch requires Zig 0.16.0. Add `zemscripten` to your
`build.zig.zon` dependencies, configure Emscripten externally, and put `emcc`
and `emrun` on `PATH`. The package does not download or activate an emsdk.

SDK discovery and a real Emscripten 6.0.6 driver invocation were exercised.
Full application/Wasm/browser acceptance remains unverified for this branch.
The binding module imports Emscripten functions and is not a native-linkable
test executable.

The obsolete activation, SDK-version and default-shell-path helpers have
been removed. Omit activation-step dependencies; pass an explicit shell
through `shell_file_path` only when your application needs one.

Add zemscripten's "root" module to your wasm compile target., then create an `emcc` build step. We use zemscripten's default flags and settings which can be overridden for your project specific requirements. Refer to the [emcc documentation](https://emscripten.org/docs/tools_reference/emcc.html). Example build.zig code:
```zig
    const wasm = b.addLibrary(.{
        .name = "MyGame",
        .linkage = .static,
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    const zemscripten = b.dependency("zemscripten", .{});
    wasm.root_module.addImport("zemscripten", zemscripten.module("root"));

    const emcc_flags = @import("zemscripten").emccDefaultFlags(b.allocator, .{
        .optimize = optimize,
        .fsanitize = false,
    });

    var emcc_settings = @import("zemscripten").emccDefaultSettings(b.allocator, .{
        .optimize = optimize,
        .emsdk_allocator = .emmalloc,
    });
    try emcc_settings.put("ALLOW_MEMORY_GROWTH", "1");

    const emcc_step = @import("zemscripten").emccStep(
        b,
        &.{ }, // src file paths 
        &.{ wasm }, // src compile steps
        .{
            .optimize = optimize,
            .flags = emcc_flags,
            .settings = emcc_settings,
            .use_preload_plugins = true,
            .embed_paths = &.{},
            .preload_paths = &.{},
            .shell_file_path = null, // set this to override the default html shell
            .js_library_path = null,
            .out_file_name = "MyGame.html", // or "MyGame.js"
            .install_dir = .{ .custom = "web" },
        },
    );

    b.getInstallStep().dependOn(emcc_step);
```

Now you can use the provided Zig panic and log overrides in your wasm's root module and define the entry point that invoked by the js output of `emcc` (by default it looks for a symbol named `main`). For example:
```zig
const std = @import("std");

const zemscripten = @import("zemscripten");
pub const panic = zemscripten.panic;

pub const std_options = std.Options{
    .logFn = zemscripten.log,
};

export fn main() c_int {
    std.log.info("hello, world.", .{});
    return 0;
}
```

You can also define a run step that invokes `emrun`. This will serve the html locally over HTTP and try to open it using your default browser. Example build.zig code:
```zig
    const html_filename = try std.fmt.allocPrint(b.allocator, "{s}.html", .{wasm.name});

    const emrun_args = .{};
    const emrun_step = @import("zemscripten").emrunStep(
        b,
        b.getInstallPath(.{ .custom = "web" }, html_filename),
        &emrun_args,
    );

    emrun_step.dependOn(emcc_step);

    b.step("emrun", "Build and open the web app locally using emrun").dependOn(emrun_step);
```
See the [emrun documentation](https://emscripten.org/docs/compiling/Running-html-files-with-emrun.html) for the difference args that can be used.
