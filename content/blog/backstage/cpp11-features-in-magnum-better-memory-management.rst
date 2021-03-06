C++11 features in Magnum: Better memory management
##################################################

:date: 2013-09-08 16:35
:modified: 2018-01-03
:archived: True
:category: Backstage
:tags: C++, Corrade, OpenGL, Qt
:summary: Regardless to what language you use, you always need to think about
    memory management. Garbage collectors might give you a sense that it's done
    automagically, but to get more performance you need to have control over
    heap usage in code running at 60 FPS. C++11 helps to avoid many heap
    allocations without sacrificing usability.

.. role:: cpp(code)
    :language: c++

.. block-primary:: Backstage

    This article is part of a series describing the *how* and *why* behind
    Magnum graphics engine. Read also the other parts:

    1.  C++11 features in Magnum: Better memory management
    2.  `C++11 features in Magnum: Simplifying daily workflow <{filename}/blog/backstage/cpp11-features-in-magnum-simplifying-daily-workflow.rst>`_
    3.  `Static and dynamic polymorphism in Magnum <{filename}/blog/backstage/static-and-dynamic-polymorphism-in-magnum.rst>`_

.. note-success:: Content care: Jan 03, 2018

    Code snippets were updated to match current state of the Magnum API,
    sections about :cpp:`std::optional` and VLAs were updated to mention
    :dox:`Containers::Optional` and the finalized versions of C++14 and C++17
    standards. Also some typos and grammar errors were fixed.

`Getting rid of explicit memory management`_
============================================

Many objects in the engine are memory-heavy (e.g. vertex data), which means
that copying them is not a good idea. Moreover, OpenGL objects (such as
textures, meshes or buffers) aren't copyable, similarly to e.g. the
:dox:`std::fstream` objects. In C++03 world it meant that you had basically two
solutions for avoiding copying. You can force the user to create an instance
beforehand and then use the function to perform some action on it:

.. code:: c++

    Trade::MeshData3D mesh;
    importer->meshData3D("steam-locomotive", mesh);

Or you can create the instance on heap, return pointer to it and instruct the
user to explicitly delete the object afterwards:

.. code:: c++

    Trade::MeshData3D* mesh = importer->meshData3D("steam-locomotive");
    if(mesh) {
        // ...
    }
    delete mesh;

Both methods require some non-trivial action from the user. In the first case
the object might have non-trivial constructor (which actually *is* the case
with :dox:`Trade::MeshData3D`) and the function has no intuitive way to tell
that the import went bad and the mesh is now invalid. The second case does heap
allocation, which is slower than stack allocation, but more importantly
requires explicit deletion, which more often than not leads to accidental
memory leaks.

C++11 introduces move semantics, which means that the object can be just moved
out from the function without copying:

.. code:: c++

    Trade::MeshData3D mesh = importer->meshData3D("steam-locomotive");

While this is finally a clean oneliner, it now isn't possible to indicate that
the mesh import failed.

The long-awaited :cpp:`std::optional`, which got finally included in C++17,
aims to solve exactly this. The optional object contains a boolean indicating
the state and additional space where the instance can be stored (i.e., without
any heap allocation). The class is inspired with :cpp:`boost::optional` but
since the engine needs to behave the same also on C++11 compilers, Corrade
introduced its own :dox:`Containers::Optional` that doesn't require C++17 and
works on C++11-capable compilers as well.

.. code:: c++

    Containers::Optional<Trade::MeshData3D> mesh = importer->meshData3D("steam-locomotive");
    if(mesh) {
        // ...
    }

Lastly, some functions return polymorphic types, which can't be done any other
way than with heap allocation. C++11's :cpp:`std::unique_ptr` will handle the
deletion implicitly and unlike :cpp:`std::shared_ptr` it adds only a tiny
overhead, because it doesn't need to do any reference counting.

.. code:: c++

    std::unique_ptr<Trade::AbstractMaterialData> material = importer->material("scratched-copper");
    if(material) {
        // ...
    }

`Initializer lists`_
====================

In C++03 code, when you want to pass a list of some values (known at
compile-time) to a function, the most performant way is this:

.. code:: c++

    Source* sources[] = {backgroundMusic, boom, laughter, eternalPain};
    Audio::Source::play(sources, 4);

If you want to write the same as one-liner, you can achieve that using
specially crafted container containing some magic with :cpp:`operator,` or
:cpp:`operator<<`, but with not exactly intuitive usage and at a cost of
run-time heap allocation, for example:

.. code:: c++

    Audio::Source::play((Array<Source*>(), backgroundMusic, boom, laughter, eternalPain));

C++11's :dox:`std::initializer_list` allows to write this as a one-liner
without any additional overhead. In many cases Magnum also provides a
:dox:`Containers::ArrayView` overload for lists of run-time dependent size.

.. code:: c++

    Audio::Source::play({backgroundMusic, boom, laughter, eternalPain});

`Variable-length arrays`_
=========================

While this feature is often frowned-upon, it has its use. Many functions in
OpenGL and other frameworks, most notably the recent :glext:`ARB_multi_bind`
extension, accept arrays of integers to do an operation on a given list of
objects. In public Magnum API this is often done using :dox:`std::initializer_list`
of pointers to given objects, as shown above. But internally the library needs
to extract IDs from all objects, put them in some newly allocated array of
variable length, pass that array to given function and then delete the array
again. Stack-allocated arrays solve this and the feature is already available
as non-standard extension in GCC, but it's not yet used in Magnum due to
possible portability issues.

.. TODO: once DynamicArray is implemented, mention it here

`Setters and move semantics`_
=============================

Setters in C++03 code commonly take const reference to object and then copy it
to the destination:

.. code:: c++

    void Configuration::setFilename(const std::string& filename) {
        _filename = filename;
    }

While taking object by reference avoids creating another copy compared to
taking object by value, it doesn't avoid unnecessary copies altogether:

.. code:: c++

    Configuration conf;
    std::string file = "game.conf";
    conf.setFilename(file);         // okay, copied from named variable
    conf.setFilename("game.conf");  // bad, copied from temporary variable

In the second case, temporary :dox:`std::string` variable is created (first
allocation), then its contents are copied (second allocation) and then this
temporary is discarded (deallocation). The unneeded allocation and deallocation
can be avoided using move semantics, but from user point-of-view the usage is
still the same. In Magnum all setters taking heavy types (strings, vectors...)
are done this way.

.. TODO: better example, this is not really perfect

.. code:: c++

    void Configuration::setFilename(std::string filename) {
        _filename = std::move(filename);
    }

`Semi-automatic memory management`_
===================================

Handling memory deallocations in inter-dependent scene graph with many shared
resources is a pain to do manually and this is exactly the case where a sane
garbage collector is actually useful. Magnum offers two ways of automatic
memory management: the scene graph and resource manager.

Scene graph is a tree of objects, similar to what Qt's :cpp:`QObject` hierarchy
is. When some object is destroyed, all its children and attached features are
automatically destroyed too. In fact, together with method chaining you can add
objects to a scene and configure them without even saving them to a variable.
You can read more about scene graph :dox:`in the documentation <scenegraph>`.

.. code:: c++

    (new Chair(&scene))
        ->translate({0.4f, 0.0f, -1.0f})
        ->rotateY(25.0_degf);

Resource manager offers more fine-grained options. Each resource stored there
can be either static (deleted on manager destruction), manually managed
(deleted on explicit :dox:`free() <ResourceManager::free()>` call, which can be
done either for particular resource type or for whole manager) or
reference-counted (deleted when last reference to the object is removed). The
behavior is more thoroguhly described :dox:`in ResourceManager class documentation <ResourceManager>`.
